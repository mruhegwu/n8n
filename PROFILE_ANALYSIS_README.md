# Performance Profile Analysis - Complete

This branch contains a comprehensive analysis of n8n's performance profiling infrastructure.

## 📚 Documentation Index

### For Decision Makers
- **[PROFILE_ANALYSIS_SUMMARY.md](./PROFILE_ANALYSIS_SUMMARY.md)** - Executive summary with key findings (5 min read)

### For Technical Deep Dive
- **[PROFILE_ANALYSIS.md](./PROFILE_ANALYSIS.md)** - Full technical analysis with implementation plans (30 min read)

### For Developers
- **[docs/PERFORMANCE_PROFILING_GUIDE.md](./docs/PERFORMANCE_PROFILING_GUIDE.md)** - How to use profiling tools and add benchmarks (15 min read)

### For Implementation
- **[.github/workflows/benchmark-microbenchmarks.yml.example](./.github/workflows/benchmark-microbenchmarks.yml.example)** - Ready-to-use CI workflow

## 🎯 Key Findings

### ✅ What's Working Well
1. **Solid Foundation:** Vitest-based microbenchmarking with proper configuration
2. **Clear Methodology:** 10% regression threshold with good detection logic
3. **Good Documentation:** Clear README explaining when and how to use benchmarks
4. **Multiple Systems:** Load testing, E2E performance, and microbenchmarks

### ❌ Critical Gap
**No CI Integration for Microbenchmarks**
- Performance regressions in expression engine go undetected
- Baselines are hardware-specific and not tracked
- No automated checks on PRs

### ⚠️ Areas for Improvement
1. Limited benchmark coverage (only expression engine)
2. No historical trend tracking
3. Missing developer profiling workflows documentation

## 🚀 Recommended Next Steps

### Phase 1: CI Integration (Week 1) - HIGH PRIORITY
Implement automated benchmark checks using the provided example workflow:
- Store baselines as GitHub Actions artifacts
- Run benchmarks on every PR touching performance-critical code
- Block PRs with >10% regressions

### Phase 2: PR Integration (Week 2)
- Post benchmark results as PR comments
- Add required status check
- Implement skip-benchmark label option

### Phase 3: Expand Coverage (Weeks 3-4)
- Add workflow traversal benchmarks
- Add node execution overhead benchmarks
- Add data transformation benchmarks

## 📊 Current Benchmarks

The system currently benchmarks the **expression engine** hot path:
- Simple property access: `={{ $json.name }}`
- Array operations: `={{ $json.items.map(i => i.value) }}`
- Complex chains: `={{ $json.items.filter(i => i.active).map(i => i.id) }}`

These benchmarks answer the critical question: "What's the baseline performance for alternative implementations (WASM, QuickJS, etc.)?"

## 💡 Quick Commands

```bash
# Run benchmarks
pnpm --filter=@n8n/performance bench

# Save baseline
pnpm --filter=@n8n/performance bench:baseline

# Check for regressions
pnpm --filter=@n8n/performance bench:ci
```

## 🔗 Related Systems

- **Microbenchmarks:** `packages/testing/performance/` (this analysis)
- **Load Testing:** `packages/@n8n/benchmark/` (k6-based)
- **E2E Performance:** `packages/testing/playwright/`

## 📈 Success Metrics

Once CI integration is complete, we'll track:
- ✅ Percentage of PRs checked for performance regressions
- ✅ Number of regressions caught before merge
- ✅ Average benchmark runtime in CI
- ✅ Coverage of critical hot paths

## 🎓 Learning Resources

1. Start with [PROFILE_ANALYSIS_SUMMARY.md](./PROFILE_ANALYSIS_SUMMARY.md) for overview
2. Read [docs/PERFORMANCE_PROFILING_GUIDE.md](./docs/PERFORMANCE_PROFILING_GUIDE.md) for hands-on usage
3. Review [PROFILE_ANALYSIS.md](./PROFILE_ANALYSIS.md) for complete technical details
4. Use [benchmark-microbenchmarks.yml.example](./.github/workflows/benchmark-microbenchmarks.yml.example) as CI starting point

## 🤝 Contributing

To add new benchmarks:
1. Follow the guide in [docs/PERFORMANCE_PROFILING_GUIDE.md](./docs/PERFORMANCE_PROFILING_GUIDE.md)
2. Ensure benchmarks measure critical hot paths
3. Include realistic test data
4. Document what question the benchmark answers

## ⏱️ Timeline Estimate

**Total effort:** 3-4 weeks for full implementation
- Week 1: CI integration (critical)
- Week 2: PR integration (important)
- Weeks 3-4: Expanded coverage + historical tracking (nice-to-have)

**Quick win:** CI integration can be done in 2-3 days for immediate value.

---

**Analysis Date:** 2026-03-17
**Branch:** `claude/analyse-profile`
**Status:** ✅ Analysis complete, ready for implementation
