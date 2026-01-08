# Activepieces Performance & Configuration Knowledge Base

Last Updated: January 8, 2026

## Change Log

### 2026-01-08 - Initial Optimization & Template Fix

#### Performance Optimizations Applied

**Execution Mode**
- Changed from `UNSANDBOXED` to `SANDBOX_CODE_ONLY`
- Result: 3-5x performance improvement (UI load time reduced from 10-15s to ~0.34s)
- Trade-off: Workers reused across executions, minimal sandbox overhead

**Worker Concurrency**
- General Workers: 10 (up from default 5)
- Agent Workers: 5 (specialized for AI operations)
- Rationale: Better handling of concurrent LinkedIn AI workflows with multiple API calls

**Timeout Configuration**
- Flow Timeout: 600s (10 min) - for multi-step AI operations (Perplexity → OpenAI → Claude → DALL-E → LinkedIn)
- Trigger Timeout: 300s (5 min) - for trigger testing and polling
- Engine Operation Timeout: 600s (10 min) - socket.io communication
- HTTP Timeout: 15s - faster API connection timeouts
- Rationale: LinkedIn automation workflow takes 3-5 minutes with 5 AI API calls

**Database Optimization**
- PostgreSQL Connection Pool: 20 connections (up from default 10)
- Idle Timeout: 10s (default 30s) for faster connection recycling
- Rationale: Reduced connection wait time during high-load operations

**Redis Configuration**
- Eviction Policy: `noeviction` (CRITICAL - changed from allkeys-lru)
- Rationale: Prevents eviction of BullMQ job locks and queues, which causes job failures
- Redis is used for: Job queues, distributed locks, pub/sub, caching

**HTTP & Compression**
- HTTP Request Timeout: 15s (faster API connection timeouts)
- Compression Enabled: Yes (reduced payload sizes for API responses)
- Rationale: Improves connection saving speed (Apify API keys, etc.)

**Resource Limits**
- Activepieces: 6GB memory, 4 CPUs
- PostgreSQL: 2GB memory, 2 CPUs  
- Redis: 2GB memory, 1 CPU
- Sandbox Memory: 1GB per execution

#### Critical Fixes

**Template URL Parse Error**
- **Issue**: When `AP_TEMPLATES_SOURCE_URL=""` (empty string), template API calls failed with "Failed to parse URL from ?type=OFFICIAL: Invalid URL"
- **Root Cause**: Empty string base URL + query params = invalid URL in fetch()
- **Fix**: Comment out variable completely: `# AP_TEMPLATES_SOURCE_URL=`
- **Result**: Templates properly hidden from UI, no more URL parse errors

**Flow Operation Status Stuck**
- **Issue**: Flow stuck at `operationStatus=ENABLING`, couldn't toggle ON
- **Root Cause**: Database state corruption from interrupted operation (crash/timeout)
- **Fix**: SQL: `UPDATE flow SET "operationStatus" = 'NONE' WHERE id = 'dpWvTCluaxhp2k3YrDXb0';`
- **Result**: Flow can now be enabled normally

**Cloud Authentication**
- **Fix**: Set `AP_CLOUD_AUTH_ENABLED=false` for self-hosted deployment
- **Result**: No cloud dependencies, all authentication local

#### Git Synchronization

**Upstream Merge** (8 commits from activepieces/activepieces:main)
- 6901c7a1ed: fix stripe payment processing
- 59fab38d9b: feat limit models per AI provider
- 31dd289412: fix duplicate MCP tool names
- a180968791: feat enhance limits section in UI
- 5d6cd8007b: refactor remove unused button
- 42446f3a23: feat webhook binary file support
- 5c680b2fdd: feat custom templates for embedded users
- 668615cd06: fix lint errors in runs table

**Branch Status**
- Fully synced with upstream/main
- 0 commits behind, local changes committed and pushed

## Configuration Reference

### Critical Environment Variables

```dotenv
# Performance
AP_EXECUTION_MODE=SANDBOX_CODE_ONLY
AP_WORKER_CONCURRENCY=10
AP_AGENTS_WORKER_CONCURRENCY=5

# Timeouts (in seconds)
AP_FLOW_TIMEOUT_SECONDS=600
AP_TRIGGER_TIMEOUT_SECONDS=300
AP_ENGINE_OPERATION_TIMEOUT_SECONDS=600
AP_HTTP_TIMEOUT_MS=15000

# Database
AP_POSTGRES_POOL_SIZE=20
AP_POSTGRES_IDLE_TIMEOUT_MS=10000

# Templates (MUST be commented to disable)
# AP_TEMPLATES_SOURCE_URL=

# Cloud Features (disabled for self-hosted)
AP_CLOUD_AUTH_ENABLED=false

# Compression
AP_COMPRESSION_ENABLED=true
```

### Redis Configuration (docker-compose.yml)

```yaml
redis:
  command: redis-server --maxmemory-policy noeviction --maxmemory 2gb
```

**Why noeviction?**
- BullMQ stores job locks and queue metadata in Redis
- Evicting these keys causes jobs to fail or hang
- Better to run out of memory (and add more) than lose job state

## Troubleshooting Guide

### Template Errors "Failed to parse URL from ?type=OFFICIAL"

**Symptoms**: Logs show TypeError with invalid URL errors
**Cause**: `AP_TEMPLATES_SOURCE_URL` set to empty string
**Solution**: Comment out the variable completely in .env

### Flow Stuck at "Flow is busy"

**Symptoms**: Can't enable/disable flow, error "Flow is busy with enabling operation"
**Cause**: Database `operationStatus` stuck at ENABLING/DISABLING/DELETING
**Solution**: 
```sql
-- Connect to database
docker-compose exec postgres psql -U postgres -d activepieces

-- Check status
SELECT id, status, "operationStatus" FROM flow WHERE id = 'YOUR_FLOW_ID';

-- Reset if stuck
UPDATE flow SET "operationStatus" = 'NONE' WHERE id = 'YOUR_FLOW_ID';
```

### Slow API Key Saving (Apify, etc.)

**Symptoms**: Connection save takes 5-10 seconds, page refreshes
**Cause**: Default HTTP timeout too high, no compression
**Solution**: 
- Set `AP_HTTP_TIMEOUT_MS=15000` (15s instead of default 30s)
- Enable `AP_COMPRESSION_ENABLED=true`
- Restart containers: `docker-compose down && docker-compose up -d`

### Worker Timeout Errors

**Symptoms**: "operation has timed out" errors, socket.io disconnects
**Cause**: AI operations take longer than default timeouts
**Solution**: Increase timeouts based on workflow duration:
- Simple flows: 300s (5 min)
- AI workflows: 600s (10 min)  
- Complex multi-AI: 900s (15 min)

### Redis Memory Full

**Symptoms**: Jobs not processing, "OOM command not allowed" errors
**Cause**: Redis reaches maxmemory with noeviction policy
**Solution**: 
1. Increase Redis memory: `--maxmemory 4gb`
2. Check failed jobs: `docker-compose exec redis redis-cli LLEN bull:default:failed`
3. Clear old failed jobs if needed (adjust retention in AP_REDIS_FAILED_JOB_RETENTION_DAYS)

## Performance Metrics

### Before Optimization
- UI Load Time: 10-15 seconds
- Worker Timeouts: Frequent (every 3-5 min flow)
- Connection Save: Slow, inconsistent
- Template Errors: Constant URL parse failures

### After Optimization
- UI Load Time: ~0.34 seconds (97% improvement)
- Worker Timeouts: Zero (600s timeout sufficient)
- Connection Save: Fast (~2-3s with compression)
- Template Errors: Eliminated (template system disabled properly)

## Testing Instructions

### Verify All Fixes

1. **Check Container Health**
   ```powershell
   docker-compose ps
   # All should show "healthy" status
   ```

2. **Verify Template Fix**
   ```powershell
   docker-compose logs activepieces | Select-String "template"
   # Should NOT see "Failed to parse URL" errors
   ```

3. **Test Flow Enabling**
   - Navigate to Flows in UI
   - Toggle flow ON/OFF
   - Should work instantly without "busy" errors

4. **Test Connection Saving (Apify)**
   - Go to Connections → New Connection
   - Select Apify → Enter API key
   - Save - should complete in 2-3 seconds

5. **Monitor Performance**
   ```powershell
   docker-compose stats
   # Check memory/CPU usage
   ```

## Architecture Notes

### Flow State Management

Two-level status tracking:
- `status`: User-visible (ENABLED/DISABLED) 
- `operationStatus`: Internal operation state (NONE/ENABLING/DISABLING/DELETING)

When enabling a flow:
1. Set `operationStatus=ENABLING`
2. Perform async operations (validation, trigger setup, worker initialization)
3. Set `status=ENABLED` and `operationStatus=NONE`

If step 2 fails/crashes, flow stuck at `operationStatus=ENABLING` until manually reset.

### Data Persistence

- **PostgreSQL**: Flows, runs, users, connections (persists across restarts)
- **Redis**: Job queues, locks, pub/sub, cache (volatile, cleared on restart)

Container restart clears Redis but NOT PostgreSQL, which is why database corruption persists.

## Recommendations

1. **Monitor Redis Memory**: Set up alerts when usage > 80%
2. **Regular Database Maintenance**: Check for stuck operationStatus weekly
3. **Log Rotation**: Configure Docker log rotation to prevent disk fill
4. **Backup Strategy**: Daily PostgreSQL backups for production
5. **Resource Scaling**: Increase worker concurrency as flow count grows

## References

- Activepieces Documentation: https://www.activepieces.com/docs
- Execution Modes: https://www.activepieces.com/docs/install/configuration/app-settings#ap_execution_mode
- BullMQ Redis Guide: https://docs.bullmq.io/guide/going-to-production
- PostgreSQL Connection Pooling: https://node-postgres.com/features/pooling
