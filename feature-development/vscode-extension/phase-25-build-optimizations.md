## Build optimizations

1. **ts-morph is bundled twice**: Once in extension.js (5.3MB) and once in worker (5.6MB)
2. **Lazy loading != Bundle size reduction**: Code is still bundled, just initialized later
3. **True bundle optimization** would require:

- Moving AnalysisEngine to worker-only communication
- Removing @vipr/engine import from extension.ts
- Using message passing for all analysis operations

## Recommendations for Future Work

**Phase 1: Worker-only Analysis (High Impact)**

- Remove AnalysisEngine from extension bundle
- Use worker communication for all analysis
- Expected reduction: ~5MB from extension bundle

**Phase 2: Code Splitting**

- Split chart.js to separate chunk (164KB)
- Load presentation/formatting code on-demand
- Expected reduction: ~200KB

**Phase 3: Tree-shaking Optimization**

- Review @vipr/common imports for unused code
- Optimize analyzer plugin imports
- Expected reduction: ~500KB
