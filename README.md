# CodePath AI301 — Open Source Contribution

**Student:** Zaid Shaikh
**CodePath Member ID:** 143945
**Course:** AI301 | AI Open Source Capstone — Summer 2026 (Section 1A)

---

## Project: lychee

- **Repository:** [lycheeverse/lychee](https://github.com/lycheeverse/lychee)
- **Fork:** [DiazSk/lychee](https://github.com/DiazSk/lychee)
- **Issue:** [#2128 — Use GitHub API for certain fragment checks](https://github.com/lycheeverse/lychee/issues/2128)
- **Language:** Rust

---

## Why I Chose This Issue

I chose issue #2128 "Use GitHub API for certain fragment checks" in [lychee](https://github.com/lycheeverse/lychee) — a fast, async link checker written in Rust — because it sits at the intersection of API integration, systems thinking, and codebase architecture, all areas I want to deepen as a data engineer who regularly builds and maintains pipelines.

I'm interested in this issue because:

1. **Architectural relevance:** The fix involves replacing naive HTTP scraping of GitHub pages with proper REST API calls. This is a pattern I understand well from my own data pipeline work — polling HTML for structured data is fragile and rate-limit-prone; using APIs is the right solution.
2. **Contained scope:** The change is scoped to one file — `lychee-lib/src/quirks/mod.rs` — which makes it manageable as a first contribution while still being a meaningful, production-impacting improvement.
3. **Active maintainers:** The maintainers (`@mre` and `@katrinafyi`) are responsive, left detailed guidance in the issue thread, and explicitly welcomed contributions from CodePath students.
4. **Learning goal:** I want hands-on experience with Rust's pattern matching and regex-based URL rewriting in a real-world codebase, and this issue gives me a clear, guided entry point into those patterns.
5. **Real-world impact:** lychee is used in CI/CD pipelines across hundreds of projects. Fixing rate-limit issues caused by unauthenticated GitHub page requests is a tangible improvement for the wider community.

From reading the issue thread and the maintainer comments, I understand the core problem: when lychee encounters a GitHub URL with a `#readme` fragment (e.g., `https://github.com/owner/repo#readme`), it currently checks it like a regular web link — hitting the GitHub HTML page directly. This causes unnecessary rate-limiting. The correct approach is to call GitHub's [Get a repository README](https://docs.github.com/en/rest/repos/contents#get-a-repository-readme) REST API endpoint instead. A `200` response means the README exists and the link is valid; the fragment check itself can be skipped.

The maintainer (`@mre`) pointed me directly to the implementation location and provided the exact regex pattern and test cases to use. PR #2194 (a prior attempt) is no longer active, so this contribution starts fresh.

Left a comment on the issue introducing myself — `@mre` confirmed the scope, pointed me to `quirks/mod.rs`, and both maintainers expressed support. `@katrinafyi` added: *"We'll help you along the way!"*

---

## Implementation Plan (Phase II Preview)

The contribution will add a new quirk in `lychee-lib/src/quirks/mod.rs` that:

1. Matches URLs of the form `github.com/{owner}/{repo}[/tree/{branch}]#readme` using the regex:
   ```
   ^https://github\.com/(?<owner>[^/]+)/(?<repo>[^/]+)(?:/tree/(?<branch>[^/]+))?#readme$
   ```
2. Rewrites the URL to the GitHub API endpoint: `https://api.github.com/repos/{owner}/{repo}/readme`
3. Returns a `200` response as success, skipping further fragment resolution

**Test cases (per maintainer guidance):**

| URL | Expected |
|-----|----------|
| `https://github.com/lycheeverse/lychee#readme` | ✅ Match |
| `https://github.com/lycheeverse/lychee/tree/main#readme` | ✅ Match |
| `https://github.com/lycheeverse/lychee/blob/main/README.md#readme` | ❌ No match |
| `https://github.com/lycheeverse/lychee#installation` | ❌ No match |

---

## Progress Log

| Phase | Status | Notes |
|-------|--------|-------|
| Phase I — Issue Selection | ✅ Complete | Issue #2128 selected, fork created, comment posted, maintainer confirmed |
| Phase II — Reproduce & Plan | 🔄 In Progress | Setting up local Rust dev environment |
| Phase III — Implementation | ⏳ Upcoming | Code changes in `quirks/mod.rs` |
| Phase IV — PR Submission | ⏳ Upcoming | Submit PR to `lycheeverse/lychee` |

---

## Links

- [lycheeverse/lychee GitHub repo](https://github.com/lycheeverse/lychee)
- [Issue #2128](https://github.com/lycheeverse/lychee/issues/2128)
- [My fork: DiazSk/lychee](https://github.com/DiazSk/lychee)
- [My comment on the issue](https://github.com/lycheeverse/lychee/issues/2128)
- [lychee CONTRIBUTING.md](https://github.com/lycheeverse/lychee/blob/master/CONTRIBUTING.md)
