# Git repository size: empirical data for threshold design

**The vast majority of Git repositories are tiny, but the distribution follows a log-normal body with a power-law tail—meaning a small fraction of repos are orders of magnitude larger than the median.** Across ~10 million repositories analyzed by the scc tool, **90% contain fewer than 300 files** and the estimated median is just 30–50 files with under 5,000 lines of code. This extreme right skew is the single most important design consideration for any algorithm classifying repositories by size: logarithmic thresholds, not linear ones, must govern your tiering logic.

The data synthesized below draws from Ben Boyter's 2019 scc analysis of 10 million repositories (the largest public code-counting study), GitHub's git-sizer tool thresholds, the `source{d}` statistical analysis of GitHub, academic papers on software metric distributions, GitHub Octoverse reports (2023–2025), and real-world benchmarks from the Linux kernel, Chromium, and Microsoft Windows monorepo.

---

## File counts cluster heavily at the low end

The most robust empirical data on file counts comes from Boyter's scc analysis of **9.1 million non-empty repositories** across GitHub, Bitbucket, and GitLab. The file-per-repo distribution is dramatically right-skewed:

| Percentile        | Files per repository |
| ----------------- | -------------------- |
| ~50th (estimated) | **30–50**            |
| ~75th (estimated) | **100–150**          |
| 85th              | **&lt;200**          |
| 90th              | **&lt;300**          |
| 95th              | **&lt;1,000**        |
| 99th+             | **>1,000**           |

The computed average is ~388 files per non-empty repo, but this number is misleading—it is inflated by a small number of extremely large repositories. The median is roughly an order of magnitude lower. **Most GitHub repositories are personal projects, homework, forks, or single-purpose utilities**; Kalliamvakou et al. (2014) found 67% of GitHub projects have a single developer and ~37% of a random sample were used for storage, experiments, or academic purposes rather than active software development.

Average code file size is approximately **19 KB** across all languages (Boyter's GopherCon data), though this varies significantly—Python files tend to be smaller than Java files by roughly 30% in mean size.

---

## Lines of code: the median repo is surprisingly small

From the same 10-million-repo dataset, aggregate statistics yield an average of **~89,800 code lines per repository** (816.8 billion total code lines across 9.1 million repos). However, the true median is estimated at **1,000–5,000 LOC**, reflecting the overwhelming dominance of small projects. The average is pulled upward by repositories in the millions-of-lines range.

For engineered software projects—those that would actually matter for your analysis tool—the distribution shifts substantially rightward. The COCOMO model and Steve McConnell's classification, widely used in software engineering, provide a practical industry-standard size taxonomy:

| Category       | Lines of code | Typical file count | Rough percentile (all GitHub) |
| -------------- | ------------- | ------------------ | ----------------------------- |
| **Trivial**    | &lt;1,000     | &lt;20             | ~0–50th                       |
| **Small**      | 1K–10K        | 20–200             | ~50–85th                      |
| **Medium**     | 10K–100K      | 200–2,000          | ~85–97th                      |
| **Large**      | 100K–1M       | 2,000–20,000       | ~97–99.5th                    |
| **Very Large** | 1M–10M        | 20,000–200,000     | ~99.5–99.99th                 |
| **Massive**    | >10M          | >200,000           | ~99.99th+                     |

The Linux kernel (~15M LOC, ~80,000 files) sits at the boundary of "Very Large" and "Massive." Chromium (~300,000 files) and the Windows monorepo (~3.5 million files) occupy the extreme tail.

---

## Commit history depth and velocity tell a consistent story

Commit counts follow a similarly heavy-tailed distribution. For the **2,500 most popular GitHub repos** (all with ≥2,150 stars), Borges et al. (2016) found:

- **Q1**: 228 commits
- **Median**: 608 commits
- **Q3**: 1,721 commits

These are elite projects; the overall GitHub median is far lower. A revealing signal: **GitHub's own code-frequency API returns errors for repositories with ≥10,000 commits**, suggesting the platform engineers expect most repos to fall well below this threshold.

For commit velocity, GitClear's analysis of 878,592 developer-years found the **median full-time developer produces ~673 commits/year** (~2.8/workday). At the platform level, GitHub saw nearly **1 billion commits in the 12 months ending August 2025**, a 25.1% year-over-year increase, across 630+ million repositories.

The `source{d}` analysis found that **commits per developer follow a power-law (polynomial) decrease**, not a log-normal—meaning a tiny fraction of developers produce an outsized share of total commits.

---

## Repository disk size correlates with all other metrics but has its own thresholds

No official "average repository size" statistic has been published by GitHub. The best available data points:

- **Top 100 most-starred projects**: median clone size of **~90 MB**
- **Most GitHub repositories**: well under **100 MB** on disk
- **Active open-source projects**: typically **10–500 MB** for the `.git` directory
- **GitHub's recommendation**: ideally **&lt;1 GB**, strongly recommended **&lt;5 GB**
- **GitLab free tier limit**: 10 GiB per project (chosen to accommodate the vast majority of projects)

Reference benchmarks for known large repositories provide the upper range:

| Repository                     | .git / clone size | Files     | Commits    |
| ------------------------------ | ----------------- | --------- | ---------- |
| **Typical active OSS project** | 10–500 MB         | 100–5,000 | 500–10,000 |
| **Linux kernel**               | ~1.5 GB / 4.6 GB  | ~80,000   | ~723,000   |
| **Chromium**                   | ~20 GB            | ~300,000  | —          |
| **Facebook monorepo**          | ~54 GB (2014)     | 6M+       | —          |
| **Windows monorepo**           | ~300 GB           | ~3.5M     | —          |

GitHub's **git-sizer** tool provides the most granular threshold framework, using a continuous "level of concern" scale. Its 1-star (first warning) thresholds are the most operationally relevant numbers for algorithm design:

| Metric                         | 1-star threshold (first concern) |
| ------------------------------ | -------------------------------- |
| Commit count                   | **500,000**                      |
| Blob (file) count              | **1,500,000**                    |
| Total blob size (uncompressed) | **10 GiB**                       |
| Files in checkout              | **50,000**                       |
| Total checkout size            | **1 GiB**                        |
| References (branches + tags)   | **25,000**                       |
| Max single blob                | **10 MiB**                       |
| Max entries per directory      | **1,000**                        |
| Max path depth                 | **10**                           |

---

## The distribution is log-normal with fat tails

Multiple academic studies converge on the same finding: **repository and file size metrics follow a log-normal distribution in the body with power-law behavior in the upper tail**. Herraiz et al. (2009) formally identified this as a **double Pareto distribution** when analyzing the Debian archive. The `source{d}` team confirmed that per-language repository size distributions are log-normal when analyzed independently, though the cross-language aggregate deviates.

This has three practical implications for algorithm design. First, **use logarithmic scales for thresholds**—boundaries at powers of 10 (or similar geometric progressions) map naturally to the data's actual structure. Second, **the tail is fatter than log-normal predicts**, so your "very large" tier needs wider headroom than naive statistical modeling would suggest. Third, **different metrics are correlated but not perfectly**—a repository can have few files but enormous individual blobs, or millions of tiny files with a modest total size. git-sizer's multi-dimensional approach reflects this reality.

Stars and forks follow a pure **power-law distribution** (only ~8% of all GitHub repositories have ever received even a single star). This means filtering by popularity dramatically changes the size distribution you'll encounter.

---

## Practical thresholds for real-time analysis vs. throttled processing

Synthesizing all data sources—Boyter's empirical percentiles, git-sizer's concern thresholds, GitHub's official guidance, and real-world performance benchmarks—here is a recommended tiering for your algorithm:

| Tier          | Files     | LOC      | .git size   | Commits   | Action                                            | % of all repos |
| ------------- | --------- | -------- | ----------- | --------- | ------------------------------------------------- | -------------- |
| **Real-time** | &lt;1,000 | &lt;100K | &lt;100 MB  | &lt;5,000 | Full background analysis, no throttling           | ~95%           |
| **Batched**   | 1K–10K    | 100K–1M  | 100 MB–1 GB | 5K–50K    | Chunked analysis, moderate throttling             | ~4%            |
| **Throttled** | 10K–100K  | 1M–10M   | 1–10 GB     | 50K–500K  | Heavy chunking, incremental processing            | ~0.9%          |
| **Deferred**  | >100K     | >10M     | >10 GB      | >500K     | On-demand only, partial analysis, sparse checkout | ~0.1%          |

The "Real-time" threshold at 1,000 files / 100K LOC / 100 MB captures **~95% of all repositories** based on Boyter's percentile data, meaning your algorithm will rarely need to fall back to slower processing modes. This is a safe and generous boundary because git operations remain fast at this scale—`git status` is instantaneous, `git log` traversal completes in under a second, and full-tree analysis is computationally trivial.

**Performance cliff points** that should inform your throttling triggers:

- **50,000+ files**: `git status` becomes noticeably slow without FSMonitor; enable `feature.manyFiles` and index v4
- **100,000+ files**: Standard operations take multiple seconds; sparse checkout becomes necessary
- **1,000,000+ files**: Git fundamentally breaks without GVFS/Scalar; Microsoft's experience with Windows showed `git status` taking hours at this scale before custom tooling
- **>1 GB .git directory**: Shallow or partial clones recommended; full clone takes minutes rather than seconds
- **>10 GB .git directory**: Full clone becomes impractical (tens of minutes to hours)

For the most robust classification algorithm, **measure multiple dimensions rather than relying on a single metric**. A repository with 500 files but a 2 GB `.git` directory (binary assets) and one with 50,000 files but 200 MB total present very different analysis challenges. The recommended approach is to compute a composite score using the logarithm of each metric, weighted by its performance impact:

```
complexity_score = w1 * log10(file_count) + w2 * log10(total_loc) +
                   w3 * log10(git_dir_bytes) + w4 * log10(commit_count)
```

where suggested starting weights are `w1=0.35, w2=0.20, w3=0.30, w4=0.15`, reflecting that file count and on-disk size are the strongest predictors of analysis time, while LOC and commit depth affect different operations.

---

## Conclusion

The critical insight for threshold design is that **Git repository metrics span roughly six orders of magnitude**—from single-file repos with a handful of commits to multi-million-file monorepos with hundreds of gigabytes of history. The log-normal-with-fat-tails distribution means logarithmic tiering is not just convenient but statistically correct. Setting your real-time analysis cutoff at the 95th percentile (~1,000 files, ~100K LOC, ~100 MB) provides a generous boundary that captures the overwhelming majority of repositories while protecting against performance degradation. The multi-dimensional nature of repository complexity—where file count, blob sizes, commit depth, and reference count independently affect different git operations—argues strongly for a composite scoring approach rather than any single-metric threshold. Finally, the distinction between "all GitHub repos" and "engineered software projects" matters enormously: if your tool targets active development projects (those with multiple contributors, CI, and regular commits), expect the median to shift roughly one tier upward from the all-repos baseline, placing more repositories in the "Batched" and "Throttled" categories than the raw percentiles suggest.
