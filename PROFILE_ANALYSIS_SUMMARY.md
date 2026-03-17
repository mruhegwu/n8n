# Performance Profiling - Quick Reference

This is a condensed reference guide. For comprehensive analysis, see [PROFILE_ANALYSIS.md](./PROFILE_ANALYSIS.md).

## Current State

### Microbenchmark System
- **Location:** `packages/testing/performance/`
- **Framework:** Vitest benchmark mode
- **Status:** ✅ Working locally, ❌ No CI integration

### Commands
```bash
# Run benchmarks
pnpm --filter=@n8n/performance bench

# Save baseline
pnpm --filter=@n8n/performance bench:baseline

# Check for regressions
pnpm --filter=@n8n/performance bench:ci
```

## Critical Gap: No CI Integration

**Problem:** Baselines are hardware-specific and not tracked in CI
**Impact:** Performance regressions in expression engine go undetected
**Risk Level:** HIGH

## Recommended Solution

Store baselines as GitHub Actions artifacts, compare against previous CI run on same runner type.

### Implementation Steps

1. **Create benchmark CI workflow:**
   ```yaml
   - name: Download previous baseline
     uses: actions/download-artifact@v3
     with:
       name: benchmark-baseline-${{ runner.os }}-${{ runner.arch }}
       path: packages/testing/performance/profiles/

   - name: Run benchmarks with regression check
     run: pnpm --filter=@n8n/performance bench:ci

   - name: Upload new baseline
     uses: actions/upload-artifact@v3
     with:
       name: benchmark-baseline-${{ runner.os }}-${{ runner.arch }}
       path: packages/testing/performance/profiles/baseline.json
   ```

2. **Add PR status check**
3. **Post results as PR comment**

## Current Benchmarks

| Category | Benchmark | Purpose |
|----------|-----------|---------|
| Hot Path | Simple property access | Baseline performance |
| Hot Path | Array map (100 items) | Typical operation |
| Hot Path | Method chain | Complex operation |
| Cold Start | Fresh workflow | Initialization cost |
| Cold Start | Reused workflow | Pooling benefit |
| Data Transfer | Small context (100 items) | Normal case |
| Data Transfer | Large context (10k items) | Stress test |

## Metrics

- **hz:** Operations per second (higher = faster)
- **Threshold:** ±10% from baseline
- **Result:** >10% slower = regression, >10% faster = improvement

## Coverage Gaps

Need benchmarks for:
- Workflow traversal operations
- Node execution overhead
- Data transformation operations
- Database query performance

## Timeline

- **Phase 1:** CI integration (Week 1)
- **Phase 2:** PR integration (Week 2)
- **Phase 3:** Historical tracking (Weeks 3-4)

## Related Systems

- **Load Testing:** `packages/@n8n/benchmark/` (k6-based)
- **E2E Performance:** `packages/testing/playwright/`
- **Nightly Benchmarks:** `.github/workflows/test-benchmark-nightly.yml`

## Next Actions

1. ✅ Analysis completed
2. ⏳ Implement CI integration
3. ⏳ Expand benchmark coverage
4. ⏳ Document developer profiling workflows

---

For detailed analysis, implementation plans, and recommendations, see [PROFILE_ANALYSIS.md](./PROFILE_ANALYSIS.md).
