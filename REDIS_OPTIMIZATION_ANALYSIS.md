# Redis Optimization Analysis for JuiceFS

## Current Performance Profile

Based on your benchmarks:
- **Redis Latency**: 0.7-0.9ms (excellent!)
- **JuiceFS Metadata Operation**: 7.12ms (10x slower than raw Redis)
- **Small File Performance**: ~108 files/s

## Root Cause Analysis

The 10x gap (0.7ms → 7.12ms) is caused by:

### 1. **Multiple Round Trips Per Operation**
Each metadata operation executes 5-10 Redis commands sequentially:
```
WATCH key        (1 RTT = 0.7ms)
GET inode        (1 RTT = 0.7ms)  
GET entry        (1 RTT = 0.7ms)
MULTI            (1 RTT = 0.7ms)
SET/RPUSH/etc    (batched)
EXEC             (1 RTT = 0.7ms)
---------------------------------
Total:           5 RTTs = 3.5ms minimum
```

### 2. **Transaction Overhead**
Every operation uses Redis transactions (WATCH/MULTI/EXEC) adding 3 extra RTTs.

### 3. **No Command Batching**
Operations are executed individually without leveraging pipelining or Lua scripts.

## Optimization Opportunities

### 🔥 **HIGH IMPACT** (50-80% improvement)

#### 1. **Replace Transactions with Lua Scripts**
```lua
-- Single RTT instead of 5-6
local script = [[
  local attr = redis.call('GET', KEYS[1])
  if not attr then return {err='ENOENT'} end
  -- Process and update
  redis.call('SET', KEYS[1], ARGV[1])
  redis.call('RPUSH', KEYS[2], ARGV[2])
  return {ok=1}
]]
```

**Files to modify**: 
- `doWrite` (line 2564)
- `doTruncate` (line 1263)
- `doSetAttr` (line 1422)

#### 2. **Batch Non-Critical Reads**
Many read operations can be pipelined:
```go
// Before: 2 sequential operations
entry := m.rdb.HGet(ctx, entryKey, name)
attr := m.rdb.Get(ctx, inodeKey)

// After: 1 pipelined operation
pipe := m.rdb.Pipeline()
entryCmd := pipe.HGet(ctx, entryKey, name)
attrCmd := pipe.Get(ctx, inodeKey)
pipe.Exec(ctx)
```

**Target methods**:
- `doLookup` (line 1064)
- `fillAttr` (already optimized with MGet)
- `doReaddir` operations

### 📈 **MEDIUM IMPACT** (20-40% improvement)

#### 3. **Optimize TxPipelined Usage**
Many places use TxPipelined for single commands (no benefit):

```go
// Line 1459 - INEFFICIENT
_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
    pipe.Set(ctx, m.inodeKey(inode), m.marshal(attr), 0)
    return nil
})

// Should be:
err = tx.Set(ctx, m.inodeKey(inode), m.marshal(attr), 0).Err()
```

#### 4. **Preload Related Data**
When we know we'll need multiple related items:
```go
// Preload all chunk metadata for small files
if attr.Length <= ChunkSize*3 {
    // Load all chunks in one go
    keys := []string{
        m.chunkKey(inode, 0),
        m.chunkKey(inode, 1),
        m.chunkKey(inode, 2),
    }
    m.rdb.MGet(ctx, keys...)
}
```

### 💡 **QUICK WINS** (10-20% improvement)

#### 5. **Connection Pool Tuning**
Your current connection pool optimization is good, but can be enhanced:
```go
// For AWS MemoryDB with 0.7ms latency
opt.PoolSize = runtime.GOMAXPROCS(0) * 30  // Increase from *20
opt.MinIdleConns = opt.PoolSize / 3        // Keep more warm connections
opt.ConnMaxIdleTime = 5 * time.Minute      // Reduce reconnection overhead
```

#### 6. **Enable Existing Scripts**
The code already has Lua scripts for lookup/resolve but they're often disabled:
```go
// Line 1070 - Scripts are only used with specific conditions
if len(m.shaLookup) > 0 && attr != nil && !m.conf.CaseInsensi && m.prefix == "" {
    // Use script
}
// Consider relaxing these conditions
```

## Implementation Priority

### Phase 1: Enable Pipelining (Already Done ✅)
- Basic pipelining infrastructure
- 2-3x improvement expected

### Phase 2: Lua Scripts for Hot Paths
Focus on most frequent operations:
1. `doWrite` - Every file write
2. `doLookup` - Every path resolution  
3. `doSetAttr` - Attribute updates

### Phase 3: Batch Read Operations
- Implement read-ahead for related data
- Use MGet/HMGet patterns more extensively

### Phase 4: Advanced Optimizations
- Redis Cluster with read replicas
- Client-side caching for immutable data
- Bloom filters for negative lookups

## Specific Code Changes Needed

### 1. Fix doWrite to use Lua Script
```go
// pkg/meta/redis.go:2564
func (m *redisMeta) doWrite(...) syscall.Errno {
    // Current: 5-6 RTTs with transaction
    // Proposed: 1 RTT with Lua script
    script := `
        local attr = redis.call('GET', KEYS[1])
        if not attr then return redis.error_reply("ENOENT") end
        -- Update attr and chunks atomically
        redis.call('SET', KEYS[1], ARGV[1])
        redis.call('RPUSH', KEYS[2], ARGV[2])
        redis.call('INCRBY', KEYS[3], ARGV[3])
        return "OK"
    `
}
```

### 2. Batch doLookup Operations
```go
// When doing multiple lookups, batch them
func (m *redisMeta) doMultiLookup(ctx Context, lookups []lookupRequest) []syscall.Errno {
    pipe := m.rdb.Pipeline()
    for _, l := range lookups {
        pipe.HGet(ctx, m.entryKey(l.parent), l.name)
    }
    // Single RTT for all lookups
    cmds, _ := pipe.Exec(ctx)
    // Process results...
}
```

### 3. Remove Unnecessary TxPipelined
Search and replace single-command TxPipelined calls with direct operations.

## Expected Results

With all optimizations:
- **Metadata latency**: 7.12ms → 1.5-2ms
- **Small file performance**: 108 files/s → 400-600 files/s
- **Overall throughput**: 3-5x improvement

## Testing Strategy

1. **Baseline**: Current performance metrics
2. **Enable pipelining**: Test with `?enable-pipelining=true`
3. **Add Lua scripts**: Measure impact per operation
4. **Full optimization**: Combined improvements

## Monitoring

Key metrics to track:
- Redis `INFO commandstats` - Commands per operation
- Redis `CLIENT LIST` - Active connections
- JuiceFS metadata operation latency histogram
- Redis network traffic (bytes/sec)
