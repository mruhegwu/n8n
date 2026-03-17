# n8n Performance Profiling Analysis

**Date:** 2026-03-17
**Branch:** `claude/analyse-profile`
**Scope:** Analysis of n8n's performance profiling and benchmarking infrastructure

## Executive Summary

This document provides a comprehensive analysis of n8n's performance profiling capabilities, focusing on the microbenchmarking infrastructure in `packages/testing/performance/` and its integration with the broader testing ecosystem.

## 1. Current State of Performance Profiling

### 1.1 Microbenchmark System

**Location:** `/packages/testing/performance/`

**Architecture:**
- **Framework:** Vitest benchmark mode
- **Configuration:** Custom settings for stable measurements
  - 1000ms duration per benchmark (2x default 500ms)
  - 100 warmup iterations (20x default 5)
  - 500ms warmup time (5x default 100ms)
- **Storage:** Profile results stored in `/profiles/` directory (gitignored)

**Key Components:**

| Component | File | Purpose |
|-----------|------|---------|
| Benchmark Runner | `vitest.config.ts` | Vitest configuration with extended measurement periods |
| Baseline Management | `scripts/save-baseline.mjs` | Saves sanitized benchmark results as baseline |
| Regression Detection | `scripts/check-regression.mjs` | Compares current vs baseline with 10% threshold |
| Expression Engine Tests | `benchmarks/expression-engine/evaluation.bench.ts` | Measures expression evaluation performance |

### 1.2 Benchmark Categories

**Hot Path Benchmarks:**
- Simple property access: `={{ $json.name }}`
- Array map operations: `={{ $json.items.map(i => i.value) }}`
- Method chains: `={{ $json.items.filter(i => i.active).map(i => i.id) }}`

**Cold Start Benchmarks:**
- Fresh workflow initialization cost
- Reused workflow performance comparison

**Data Transfer Benchmarks:**
- Small context (100 items)
- Large context (10,000 items)

**Purpose:** These benchmarks answer critical questions about expression engine performance, particularly relevant for evaluating alternative implementations (WASM sandbox, QuickJS, etc.).

### 1.3 Regression Detection Mechanism

**Threshold:** 10% performance degradation
- **>10% slower:** CI fails (regression)
- **>10% faster:** CI passes, suggests baseline update

**Workflow:**
```bash
# 1. Save baseline before changes
pnpm --filter=@n8n/performance bench:baseline

# 2. Make code changes

# 3. Check for regressions
pnpm --filter=@n8n/performance bench:ci
```

**Output Format:**
```
Benchmark Comparison (±10% threshold)
──────────────────────────────────────────────────────────────────
❌ operation name                     45000 hz (was  50000)  -10.0%
✅ operation name                     55000 hz (was  50000)  +10.0%
🆕 new operation                      60000 hz (was     N/A)    new
```

## 2. CI Integration Analysis

### 2.1 Load Testing Infrastructure

**Location:** `packages/@n8n/benchmark/`
**Type:** k6-based load testing

**Workflows:**
- `test-benchmark-nightly.yml` - Nightly cloud-based benchmarks
- `test-e2e-performance-reusable.yml` - E2E performance testing
- `build-benchmark-image.yml` - Docker image building

**Configuration:**
- Azure cloud provisioning via Terraform
- 5 virtual users
- 1 minute duration
- Results posted to webhook

### 2.2 Microbenchmark CI Gap (TODO)

**Current Issue:**
> Baselines are hardware-specific (an 8-core MacBook baseline is meaningless on a 2-core runner)

**Problem:** The microbenchmark system (`packages/testing/performance/`) has no CI integration yet. Baselines in `/profiles/` are gitignored and machine-specific.

**Proposed Solutions:**

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| **A** | Store baselines as CI artifacts | Simple, uses existing infrastructure | Per-runner-type complexity |
| **B** | External storage (S3, benchmark service) | Centralized, accessible | Additional infrastructure |
| **C** | Compare against previous CI run | Relative comparisons | Requires consistent runner types |

## 3. Profile Data Structure

### 3.1 Benchmark Results Format

**File:** `profiles/benchmark-results.json` (generated)
**File:** `profiles/baseline.json` (sanitized for comparison)

**Structure:**
```json
{
  "files": [
    {
      "filepath": "benchmarks/expression-engine/evaluation.bench.ts",
      "groups": [
        {
          "fullName": "Hot Path",
          "benchmarks": [
            {
              "name": "simple property access",
              "hz": 50000,
              "mean": 0.02,
              "p99": 0.05,
              "rme": 0.5,
              "samples": 10000
            }
          ]
        }
      ]
    }
  ]
}
```

**Metrics:**
- `hz`: Operations per second (higher = faster)
- `mean`: Average time per operation (ms)
- `p99`: 99th percentile latency (ms)
- `rme`: Relative margin of error (%)
- `samples`: Number of iterations

### 3.2 Path Sanitization

The `save-baseline.mjs` script sanitizes absolute paths to make baselines portable:
```javascript
file.filepath = file.filepath.replace(/^.*\/benchmarks\//, 'benchmarks/');
```

This allows baselines to be:
- Compared across machines (if hardware-matched)
- Potentially committed to git (if CI integration implemented)

## 4. Performance Monitoring Ecosystem

### 4.1 Multiple Profiling Systems

n8n has **two distinct** performance measurement systems:

| System | Type | Purpose | Location |
|--------|------|---------|----------|
| Microbenchmarks | Code-level | Hot path regression detection | `packages/testing/performance/` |
| Load Testing | System-level | Full workflow throughput | `packages/@n8n/benchmark/` |

**Key Difference:**
- **Microbenchmarks:** Measure individual function performance (expression evaluation)
- **Load Testing:** Measure full system performance under load (workflow execution)

### 4.2 Playwright Performance Tests

**Location:** `packages/testing/playwright/`

**E2E Performance Tests:**
- Workflow execution timing
- UI interaction latency
- Network request performance

**Integration:** The `test-e2e-performance-reusable.yml` workflow runs Playwright tests with performance assertions.

## 5. Gap Analysis and Recommendations

### 5.1 Critical Gaps

#### 1. **No CI Integration for Microbenchmarks**
- **Impact:** HIGH
- **Risk:** Performance regressions in expression engine go undetected
- **Recommendation:** Implement Option C (compare against previous CI run)
  - Store baseline per runner type as GitHub Actions artifact
  - Download previous run's baseline before comparison
  - Fail PR if >10% regression detected

#### 2. **Limited Benchmark Coverage**
- **Impact:** MEDIUM
- **Current:** Only expression engine benchmarked
- **Missing:**
  - Workflow traversal operations
  - Node execution overhead
  - Data transformation operations
  - Database query performance
- **Recommendation:** Add benchmarks for workflow engine hot paths

#### 3. **No Historical Trend Tracking**
- **Impact:** MEDIUM
- **Current:** Only point-in-time comparisons
- **Missing:** Long-term performance trend visualization
- **Recommendation:** Store benchmark results with timestamps for trend analysis

#### 4. **Baseline Management Unclear**
- **Impact:** LOW
- **Current:** Manual baseline updates
- **Missing:** Clear policy on when to update baselines
- **Recommendation:** Document baseline update policy (e.g., after intentional optimizations)

### 5.2 Strengths

1. **Robust Measurement Configuration:**
   - Extended warmup periods ensure JIT compilation is complete
   - Longer benchmark durations provide stable results

2. **Clear Regression Threshold:**
   - 10% threshold is well-balanced (catches real issues, avoids noise)

3. **Good Documentation:**
   - `README.md` clearly explains when to use benchmarks vs other tools

4. **Separation of Concerns:**
   - Microbenchmarks for code-level performance
   - Load tests for system-level performance
   - Playwright for E2E performance

## 6. Implementation Plan for CI Integration

### Phase 1: Baseline Artifact Management (Week 1)

**Goal:** Store and retrieve baselines in CI

**Tasks:**
1. Create GitHub Actions workflow for microbenchmark CI
2. Store baseline as workflow artifact keyed by runner type
3. Download previous baseline before running regression check
4. Upload new baseline as artifact after successful run

**Acceptance Criteria:**
- Benchmarks run on every PR
- Baselines persist across CI runs
- Regression check compares against previous CI run on same runner type

### Phase 2: PR Integration (Week 2)

**Goal:** Block PRs with performance regressions

**Tasks:**
1. Add required status check for benchmark CI
2. Post benchmark results as PR comment
3. Highlight regressions in red, improvements in green
4. Add "skip-benchmark" label option for non-performance PRs

**Acceptance Criteria:**
- PRs cannot merge with >10% regression
- Developers see benchmark results in PR comments
- Clear override mechanism for intentional regressions

### Phase 3: Historical Tracking (Week 3-4)

**Goal:** Visualize performance trends over time

**Tasks:**
1. Store benchmark results with git SHA and timestamp
2. Create dashboard for historical trend visualization
3. Alert on sustained degradation across multiple PRs
4. Publish performance metrics to Posthog or similar

**Acceptance Criteria:**
- Performance trends visible per metric
- Alerts trigger when sustained degradation detected
- Metrics integrated with existing monitoring

## 7. Code Quality Assessment

### 7.1 Benchmarking Code

**File:** `benchmarks/expression-engine/evaluation.bench.ts`

**Quality:** HIGH
- Clear naming conventions
- Realistic test data
- Good separation of setup vs measurement
- Comprehensive coverage of use cases

**Strengths:**
- Answers specific questions (noted in comments)
- Uses shared workflow instance to simulate production
- Tests both cold start and hot path scenarios

**Recommendations:**
- Add more edge cases (empty arrays, null values)
- Test error handling performance
- Benchmark complex nested expressions

### 7.2 Infrastructure Scripts

**Files:** `scripts/save-baseline.mjs`, `scripts/check-regression.mjs`

**Quality:** HIGH
- Clear error handling
- Good user feedback (emoji + color)
- Proper exit codes for CI integration
- Path sanitization for portability

**Strengths:**
- Simple, focused responsibilities
- Easy to understand and maintain
- Good logging for debugging

**Minor Issues:**
- No logging level configuration
- Could use more detailed regression report (e.g., which operations regressed)

## 8. Alternative Profiling Tools Evaluation

### 8.1 Node.js Built-in Profiler

**Tool:** `node --prof` + `node --prof-process`

**Use Case:** Deep profiling of specific workflows
**Status:** Not integrated

**Recommendation:** Add helper script for developers to profile specific workflows locally.

### 8.2 Chrome DevTools Profiler

**Tool:** `--inspect` flag + Chrome DevTools

**Use Case:** Interactive profiling during development
**Status:** Not documented

**Recommendation:** Add documentation on how to profile n8n workflows with Chrome DevTools.

### 8.3 Clinic.js

**Tool:** `clinic flame`, `clinic doctor`, `clinic bubbleprof`

**Use Case:** Comprehensive Node.js performance analysis
**Status:** Not integrated

**Recommendation:** Evaluate for deep dive investigations, not routine monitoring.

## 9. Metrics and KPIs

### 9.1 Current Metrics

**Expression Engine:**
- Simple property access: Target >50,000 hz
- Array map (100 items): Target >10,000 hz
- Method chain: Target >5,000 hz

### 9.2 Recommended Additional Metrics

**Workflow Engine:**
- Node execution overhead: <1ms per node
- Workflow traversal: <0.1ms per node
- Connection resolution: <0.01ms per connection

**Data Processing:**
- JSON parsing: >100,000 items/sec
- Data transformation: >50,000 items/sec

**API Performance:**
- Workflow list: <100ms for 1000 workflows
- Workflow load: <50ms for typical workflow
- Node type registry: <1ms per lookup

## 10. Conclusion

The n8n performance profiling infrastructure demonstrates strong technical foundations with well-designed microbenchmarking capabilities. The primary gap is **CI integration**, which is critical for preventing performance regressions in production.

**Key Takeaways:**

1. ✅ **Solid Foundation:** Vitest-based benchmarking with appropriate configuration
2. ✅ **Clear Methodology:** Well-documented regression detection approach
3. ❌ **Missing CI Integration:** Critical gap preventing automated regression detection
4. ⚠️ **Limited Coverage:** Only expression engine benchmarked, needs expansion

**Priority Actions:**

1. **Implement CI integration** for microbenchmarks (Phase 1 plan above)
2. **Expand benchmark coverage** to workflow engine hot paths
3. **Document profiling workflows** for developers

**Timeline:** With focused effort, CI integration can be completed in 2-3 weeks, significantly improving n8n's ability to detect and prevent performance regressions.

## Appendix A: Command Reference

```bash
# Microbenchmarks
pnpm --filter=@n8n/performance bench          # Run benchmarks
pnpm --filter=@n8n/performance bench:baseline # Save baseline
pnpm --filter=@n8n/performance bench:ci       # CI check

# Load Testing
pnpm --filter=@n8n/benchmark provision-cloud-env  # Provision Azure
pnpm --filter=@n8n/benchmark benchmark-in-cloud   # Run load test
pnpm --filter=@n8n/benchmark destroy-cloud-env    # Cleanup

# E2E Performance
pnpm --filter=n8n-playwright test:local       # Run E2E tests
```

## Appendix B: File Locations

```
packages/
├── testing/
│   ├── performance/           # Microbenchmarks (this analysis focus)
│   │   ├── benchmarks/
│   │   │   └── expression-engine/
│   │   │       └── evaluation.bench.ts
│   │   ├── profiles/          # Results storage (gitignored)
│   │   │   ├── baseline.json
│   │   │   └── benchmark-results.json
│   │   ├── scripts/
│   │   │   ├── save-baseline.mjs
│   │   │   └── check-regression.mjs
│   │   ├── package.json
│   │   ├── vitest.config.ts
│   │   └── README.md
│   └── playwright/            # E2E performance tests
└── @n8n/
    └── benchmark/             # Load testing infrastructure
        └── (k6-based)
```

## Appendix C: Related Workflows

```
.github/workflows/
├── test-benchmark-nightly.yml          # Cloud load testing
├── test-benchmark-destroy-nightly.yml  # Cleanup
├── test-e2e-performance-reusable.yml   # Playwright performance
└── build-benchmark-image.yml           # Docker image building
```

---

**Document Version:** 1.0
**Last Updated:** 2026-03-17
**Maintainer:** Development Infrastructure Team
