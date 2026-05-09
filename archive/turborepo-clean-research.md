# Turborepo Clean Research

This document summarizes research into Turborepo's official best practices for cleaning monorepos and the rationale behind our implementation.

## Research Summary

### Official Turborepo Documentation

As of Turborepo 2.8.x (February 2026), the official documentation provides:

- **No built-in `turbo clean` command** - Verified through CLI help and API reference
- **Cache management** - Documented via `--force` flag to bypass cache
- **Pruning capabilities** - `turbo prune` for creating deployment subsets
- **Output configuration** - `outputs` array in `turbo.json` defines what gets cached

Source: [Turborepo Documentation](https://turborepo.dev/docs)

### Community Proposals

The `turbo clean` feature has been proposed in the Turborepo community:

- [GitHub Discussion 10552](https://github.com/vercel/turborepo/discussions/10552) - Feature request for `turbo clean:TASK` command
- Proposed syntax: `turbo clean:build`, `turbo clean:*`
- Goal: Eliminate cached artifacts while referencing `outputs` in `turbo.json`
- Status: Open discussion, not yet implemented

### Best Practices from Ecosystem

From research across Turborepo implementations with pnpm:

1. **Define outputs explicitly** in `turbo.json` for proper caching
2. **Package-level clean scripts** - Each package defines its own cleanup
3. **Manual cache cleanup** - Delete `.turbo` directories when needed
4. **Cross-platform compatibility** - Use platform-agnostic tools where possible

Sources:

- [How we configured pnpm and Turborepo for our monorepo - Nhost](https://nhost.io/blog/how-we-configured-pnpm-and-turborepo-for-our-monorepo)
- [Building a Monorepo with pnpm and Turborepo - Medium](https://vinayak-hegde.medium.com/building-a-monorepo-with-pnpm-and-turborepo-a-journey-to-efficiency-cfeec5d182f5)

## Our Previous Implementation

The previous root clean script was:

```bash
turbo clean && rm -rf node_modules .turbo && \
  find . -name 'tsconfig.tsbuildinfo' -type f -delete && \
  find . -type d -name '.turbo' -not -path '*/node_modules/*' -exec rm -rf {} + 2>/dev/null || true
```

### What It Did

1. Ran `turbo clean` - Execute each package's clean script
2. Removed root `node_modules` and `.turbo`
3. Deleted all `tsconfig.tsbuildinfo` files
4. Recursively removed `.turbo` directories (with error suppression)

### Problems Identified

1. **Command length** - Complex one-liner difficult to maintain
2. **No dry-run mode** - Could not preview changes before execution
3. **Silent errors** - `2>/dev/null || true` hid potential issues
4. **No feedback** - Users could not see what was being cleaned
5. **Not aligned with patterns** - Other scripts in `/scripts/` directory use separate files

## New Implementation

### Design Decisions

1. **Separate script file** - Follows pattern of `compile-docs.mjs` and `verify-build-order.sh`
2. **Bash over Node.js** - File operations better suited to shell scripting
3. **Verbose output** - Clear feedback on what's being cleaned
4. **Dry-run mode** - Safe preview with `--dry-run` flag
5. **Step-by-step execution** - Logical phases with clear descriptions
6. **Error handling** - Explicit checks rather than silent suppression

### Script Structure

```bash
scripts/clean.sh
```

Executes in four steps:

1. Package-level cleaning via `turbo clean`
2. Turborepo cache removal (`.turbo` directories)
3. TypeScript build info cleanup (`tsconfig.tsbuildinfo`)
4. Root dependencies removal (`node_modules`)

### Advantages Over Previous Approach

- **Maintainability** - Easy to read and modify
- **Safety** - Dry-run mode prevents accidents
- **Visibility** - Users see exactly what gets deleted
- **Documentation** - Inline comments explain each step
- **Debugging** - Verbose output helps troubleshoot issues
- **Consistency** - Follows repository script patterns

## Alignment with Turborepo Best Practices

Our implementation aligns with Turborepo best practices:

1. **Task-based outputs** - Each package defines `outputs` in `turbo.json`
2. **Package autonomy** - Clean scripts respect package boundaries
3. **Cache strategy** - Explicit cache directory management
4. **No assumptions** - Clean what we define, not what we assume

### Turborepo Cache Strategy

Turborepo's caching works by:

- Hashing task inputs (source files, environment, dependencies)
- Storing outputs in `.turbo/cache` directories
- Invalidating cache when any input changes
- Providing both local and remote caching

Our clean script respects this by:

- Running package clean scripts first (via `turbo clean`)
- Explicitly removing cache directories
- Not interfering with Turborepo's cache logic during normal operation

## When Turborepo Adds `turbo clean`

If Turborepo implements an official `turbo clean` command:

1. **Evaluate functionality** - Compare with our implementation
2. **Consider migration** - Assess benefits vs current approach
3. **Maintain compatibility** - May need to keep custom script for additional cleanup
4. **Update documentation** - Reference official command where appropriate

The current implementation provides flexibility to add custom cleanup logic beyond what an official command might provide.

## Cache Performance Insights

From Turborepo documentation and community research:

- Cache hits reduce 30-second builds to 0.2 seconds
- Proper `outputs` configuration is critical for cache correctness
- Cache correctness beats cache speed
- Force re-execution with `--force` flag when needed

Our clean script is designed for scenarios where you want a completely fresh start, not just cache invalidation.

## Conclusion

The Vipr monorepo clean implementation:

- Follows Turborepo ecosystem best practices
- Provides functionality not yet available in Turborepo core
- Maintains consistency with existing repository patterns
- Offers safety features (dry-run) and visibility (verbose output)
- Aligns with the proposed `turbo clean` feature concept

As Turborepo evolves, we can adapt our implementation while maintaining the benefits of our custom approach.

## References

- [Turborepo Caching Documentation](https://turborepo.dev/docs/crafting-your-repository/caching)
- [Turborepo API Reference](https://turborepo.dev/docs/reference)
- [turbo clean Feature Request](https://github.com/vercel/turborepo/discussions/10552)
- [Cache Removal Discussion](https://github.com/vercel/turborepo/discussions/5270)
- [pnpm Workspace Documentation](https://pnpm.io/workspaces)
- [Nhost Turborepo Configuration](https://nhost.io/blog/how-we-configured-pnpm-and-turborepo-for-our-monorepo)
