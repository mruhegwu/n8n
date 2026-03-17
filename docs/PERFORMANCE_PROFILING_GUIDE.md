# Performance Profiling Developer Guide

This guide helps developers profile performance issues and add benchmarks to n8n.

## Quick Start

### Running Existing Benchmarks

```bash
# Run all benchmarks
pnpm --filter=@n8n/performance bench

# Save results as baseline
pnpm --filter=@n8n/performance bench:baseline

# Check for regressions against baseline
pnpm --filter=@n8n/performance bench:ci
```

## When to Add Benchmarks

✅ **DO benchmark:**
- Hot paths executed thousands of times per workflow
- Expression evaluation and data transformation
- Core workflow engine operations
- Operations running millions of times per day across all users

❌ **DON'T benchmark:**
- API endpoint latency (use load testing instead)
- Database queries (use query analysis tools)
- Frontend rendering (use Chrome DevTools)
- One-off operations (startup, migrations)

**Rule of thumb:** If it runs millions of times per day, benchmark it.

## Adding a New Benchmark

### 1. Create a benchmark file

```typescript
// packages/testing/performance/benchmarks/my-feature/operation.bench.ts
import { bench, describe } from 'vitest';
import { MyClass } from 'my-package';

// Setup outside bench functions (not measured)
const testData = {
  small: Array(100).fill(0).map((_, i) => ({ id: i, value: i * 10 })),
  large: Array(10000).fill(0).map((_, i) => ({ id: i, value: i * 10 })),
};

const instance = new MyClass();

// Optional: Warmup to ensure JIT compilation is complete
for (let i = 0; i < 1000; i++) {
  instance.process(testData.small);
}

describe('My Feature', () => {
  bench('process small dataset', () => {
    instance.process(testData.small);
  });

  bench('process large dataset', () => {
    instance.process(testData.large);
  });
});
```

### 2. Run your benchmark

```bash
pnpm --filter=@n8n/performance bench
```

### 3. Save as baseline

```bash
pnpm --filter=@n8n/performance bench:baseline
```

### 4. Make changes and check for regressions

```bash
# Make your code changes...

# Check if performance degraded
pnpm --filter=@n8n/performance bench:ci
```

## Understanding Benchmark Results

```
name                      hz      min    max   mean    p99    rme   samples
simple property access  50,000   0.01   0.15   0.02   0.05  ±0.8%   25000
```

| Metric | Meaning | Goal |
|--------|---------|------|
| **hz** | Operations per second | Higher = faster |
| **mean** | Average time per operation (ms) | Lower = faster |
| **p99** | 99th percentile latency (ms) | Lower = more consistent |
| **rme** | Relative margin of error | Lower = more reliable |
| **samples** | Number of iterations | More = more reliable |

## Regression Detection

- **>10% slower:** ❌ Regression (CI fails)
- **±10%:** ✅ Within threshold (CI passes)
- **>10% faster:** ✅ Improvement (consider updating baseline)

## Profiling Tools for Deep Dives

### Option 1: Node.js Built-in Profiler

Best for CPU profiling:

```bash
# Run n8n with profiling
node --prof packages/cli/bin/n8n start

# Process the profile
node --prof-process isolate-*.log > profile.txt

# Analyze profile.txt to find bottlenecks
```

### Option 2: Chrome DevTools

Best for interactive profiling:

```bash
# Run n8n with inspector
node --inspect packages/cli/bin/n8n start

# Open chrome://inspect in Chrome
# Click "inspect" under n8n process
# Go to Profiler tab and take CPU profile
```

### Option 3: Clinic.js

Best for comprehensive analysis:

```bash
# Install clinic.js
npm install -g clinic

# Run with different profilers
clinic doctor -- node packages/cli/bin/n8n start
clinic flame -- node packages/cli/bin/n8n start
clinic bubbleprof -- node packages/cli/bin/n8n start
```

## Best Practices

### 1. Keep Benchmarks Focused

❌ **Bad:** Testing entire workflow execution
```typescript
bench('execute workflow', () => {
  workflow.execute(); // Too broad, many variables
});
```

✅ **Good:** Testing specific operation
```typescript
bench('evaluate expression', () => {
  workflow.expression.evaluate('={{ $json.name }}', data);
});
```

### 2. Use Realistic Data

❌ **Bad:** Artificial data
```typescript
const data = [{ json: { value: 1 } }];
```

✅ **Good:** Representative data
```typescript
const data = [{
  json: {
    id: '123',
    name: 'Test User',
    email: 'test@example.com',
    items: Array(100).fill(0).map((_, i) => ({
      id: i,
      value: i * 10,
      active: i % 2 === 0
    }))
  }
}];
```

### 3. Separate Setup from Measurement

❌ **Bad:** Setup inside benchmark
```typescript
bench('operation', () => {
  const instance = new MyClass(); // Setup measured!
  instance.process(data);
});
```

✅ **Good:** Setup outside benchmark
```typescript
const instance = new MyClass(); // Setup not measured

bench('operation', () => {
  instance.process(data);
});
```

### 4. Add Warmup for JIT-sensitive Code

```typescript
const instance = new MyClass();

// Warmup: Run 1000 times to trigger JIT optimization
for (let i = 0; i < 1000; i++) {
  instance.process(smallData);
}

describe('My Feature', () => {
  // Now benchmarks measure hot path, not JIT compilation
  bench('process', () => {
    instance.process(smallData);
  });
});
```

## Interpreting Performance Issues

### High Variance (High RME)

**Symptom:** `rme: ±15%` or higher

**Causes:**
- Background processes interfering
- Garbage collection pauses
- External dependencies (network, disk)

**Solutions:**
- Close other applications
- Run multiple times to verify
- Increase warmup iterations
- Mock external dependencies

### Unexpectedly Low Hz

**Symptom:** Much lower operations/sec than expected

**Causes:**
- Setup code inside benchmark
- Missing warmup for JIT compilation
- Synchronous I/O operations
- Expensive object creation

**Solutions:**
- Move setup outside bench function
- Add warmup iterations
- Mock I/O operations
- Reuse objects when possible

## Common Scenarios

### Scenario 1: Comparing Two Implementations

```typescript
import { bench, describe } from 'vitest';
import { currentImpl, newImpl } from './my-feature';

const testData = createTestData();

describe('Implementation Comparison', () => {
  bench('current implementation', () => {
    currentImpl(testData);
  });

  bench('new implementation', () => {
    newImpl(testData);
  });
});
```

### Scenario 2: Testing Scalability

```typescript
const sizes = [10, 100, 1000, 10000];

for (const size of sizes) {
  const data = Array(size).fill(0).map((_, i) => ({ id: i }));

  describe(`Scalability: ${size} items`, () => {
    bench(`process ${size} items`, () => {
      instance.process(data);
    });
  });
}
```

### Scenario 3: Measuring Optimization Impact

```bash
# 1. Save baseline before optimization
pnpm --filter=@n8n/performance bench:baseline

# 2. Make optimization changes
# ... edit code ...

# 3. Check improvement
pnpm --filter=@n8n/performance bench:ci

# 4. If improved >10%, update baseline
pnpm --filter=@n8n/performance bench:baseline
```

## Troubleshooting

### "No baseline found" Error

```bash
# Create initial baseline
pnpm --filter=@n8n/performance bench:baseline
```

### Benchmarks Taking Too Long

The vitest config runs benchmarks for 1000ms each with 100 warmup iterations. For faster iteration during development:

```typescript
// Temporarily reduce in vitest.config.ts
benchmark: {
  time: 100,           // Faster, less reliable
  warmupIterations: 10 // Faster warmup
}
```

Remember to revert before committing!

### Results Inconsistent Between Runs

This is normal. Hardware-specific factors affect results:
- CPU temperature/throttling
- Background processes
- Memory pressure
- JIT optimization timing

**Solutions:**
- Run benchmarks multiple times
- Close other applications
- Use consistent hardware for comparisons
- Focus on large differences (>10%)

## CI Integration (Coming Soon)

Once CI integration is implemented:

1. Benchmarks will run automatically on PRs
2. Results will be posted as PR comments
3. PRs will be blocked if performance regresses >10%
4. Baselines will be stored per runner type

## Resources

- [Vitest Benchmark Docs](https://vitest.dev/guide/features.html#benchmarking-experimental)
- [Main Analysis Document](./PROFILE_ANALYSIS.md)
- [Quick Reference](./PROFILE_ANALYSIS_SUMMARY.md)
- Example benchmarks: `packages/testing/performance/benchmarks/expression-engine/evaluation.bench.ts`

## Getting Help

1. Check existing benchmarks for patterns
2. Review this guide and main analysis document
3. Ask in #dev-performance Slack channel
4. Create GitHub issue with `performance` label
