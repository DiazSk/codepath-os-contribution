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
- **Working Branch:** [fix/issue-2128-readme-api](https://github.com/DiazSk/lychee/tree/fix/issue-2128-readme-api)

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

## Solution Plan

The contribution adds a new quirk in `lychee-lib/src/quirks/mod.rs`. Below is the step-by-step engineering roadmap derived from maintainer directives.

### Step 1 — Regex Detection

Compile the following pattern (via `once_cell` or `lazy_static`) to detect eligible GitHub `#readme` URLs:

```
^https://github\.com/(?<owner>[^/]+)/(?<repo>[^/]+)(?:/tree/(?<branch>[^/]+))?#readme$
```

This matches both `github.com/{owner}/{repo}#readme` and `github.com/{owner}/{repo}/tree/{branch}#readme`, and rejects anything that is not a root-level `#readme` fragment (e.g., blob paths, other fragments).

**Test cases (per maintainer guidance):**

| URL | Expected |
|-----|----------|
| `https://github.com/lycheeverse/lychee#readme` | ✅ Match |
| `https://github.com/lycheeverse/lychee/tree/main#readme` | ✅ Match |
| `https://github.com/lycheeverse/lychee/blob/main/README.md#readme` | ❌ No match |
| `https://github.com/lycheeverse/lychee#installation` | ❌ No match |

### Step 2 — URL Rewrite

Extract the `owner` and `repo` named captures from the regex match and construct the GitHub REST API endpoint:

```
https://api.github.com/repos/{owner}/{repo}/readme
```

The `branch` capture is intentionally discarded — the default README endpoint returns the repository's default branch README, which is the semantically correct target for a `#readme` fragment check.

### Step 3 — HTTP Request via lychee's Existing Client

Use lychee's existing `GITHUB_TOKEN`-aware HTTP `Client` (already injects auth headers when the token is present) — no new HTTP client or `octocrab` dependency is needed, keeping build times low per `@katrinafyi`'s directive.

Issue a **`GET` request** (not `HEAD`) to the API URL. GitHub's `/readme` endpoint is optimized for GET and returns lightweight metadata JSON; HEAD may return inconsistent status codes depending on repository visibility.

The request must include the header:

```
Accept: application/vnd.github.v3+json
```

### Step 4 — 200 OK Bypasses Fragment Checking

If the API returns `200 OK`, return `Status::Ok` immediately and **do not** proceed to HTML fragment resolution. A 200 from the README endpoint is sufficient proof that the `#readme` link is valid — the fragment itself names the README resource, not an anchor within a page.

Non-200 responses (404 for missing repo, 403 for private/forbidden) propagate as link failures through lychee's normal error path.

### Step 5 — Unit Tests

Add unit tests in `quirks/mod.rs` covering the four maintainer-specified match cases above. Existing test cases must remain intact.

---

## Reproduction Steps

### Prerequisites

- Rust toolchain installed (`rustup`, stable channel)
- lychee cloned from upstream

### 1. Clone the upstream repo

```bash
git clone https://github.com/lycheeverse/lychee.git
cd lychee
```

### 2. Create a test input file

```bash
cat > /tmp/test-readme-links.md << 'EOF'
# Test: GitHub #readme fragment links

- [lychee readme](https://github.com/lycheeverse/lychee#readme)
- [lychee main branch readme](https://github.com/lycheeverse/lychee/tree/main#readme)
- [non-readme fragment](https://github.com/lycheeverse/lychee#installation)
EOF
```

### 3. Build and run lychee with verbose output

```bash
cargo build --release -p lychee
./target/release/lychee \
  --verbose \
  --no-progress \
  /tmp/test-readme-links.md
```

**What to observe (current behavior):** Run without a `GITHUB_TOKEN` to force the rate-limit issue quickly — GitHub returns HTTP 429 or 403 after a handful of unauthenticated requests against the HTML pages. The `--verbose` logs show lychee fetching the full GitHub HTML page (`https://github.com/lycheeverse/lychee`) and attempting to locate the `#readme` anchor in the response body. Even when authenticated, the verbose logs prove lychee is downloading full HTML payloads instead of calling the lightweight API JSON endpoint. With the fix applied, the same URLs instead call `https://api.github.com/repos/lycheeverse/lychee/readme` (with `Accept: application/vnd.github.v3+json`) and return immediately on `200 OK`.

---

## Implementation Summary

**File changed:** [`lychee-lib/src/quirks/mod.rs`](https://github.com/DiazSk/lychee/blob/fix/issue-2128-readme-api/lychee-lib/src/quirks/mod.rs)

The change adds one static regex and one new `Quirk` struct to `Quirks::default()` — no new files, no new dependencies, no new imports (all required types were already in scope).

### What was added

**Static regex pattern** (inserted after the existing `GITHUB_BLOB_*` patterns):

```rust
static GITHUB_README_PATTERN: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^https://github\.com/(?<owner>[^/]+)/(?<repo>[^/]+)(?:/tree/(?<branch>[^/]+))?#readme$")
        .unwrap()
});
```

**New quirk** (appended to `Quirks::default()`):

```rust
Quirk {
    name: "check GitHub README existence via API",
    pattern: &GITHUB_README_PATTERN,
    rewrite: |mut request, captures| {
        let owner = captures.name("owner").unwrap().as_str();
        let repo = captures.name("repo").unwrap().as_str();
        *request.url_mut() = Url::parse(
            &format!("https://api.github.com/repos/{owner}/{repo}/readme")
        ).unwrap();
        request
            .headers_mut()
            .insert(header::ACCEPT, HeaderValue::from_static("application/vnd.github.v3+json"));
        request
    },
},
```

The `branch` capture is intentionally discarded — the GitHub REST API returns the default-branch README, which is semantically correct for a root-level `#readme` fragment. The fragment bypass is implicit: by rewriting to an API URL that carries no fragment, lychee's downstream fragment checker never receives one to resolve.

### Testing Notes

**Run the quirk unit tests:**

```bash
cargo test -p lychee-lib -- quirks
```

**New tests added** (all inside `mod tests` in `quirks/mod.rs`):

| Test | What it verifies |
|------|-----------------|
| `test_github_readme_pattern` (rstest, 4 cases) | Regex matches/rejects the correct URL shapes |
| `test_github_readme_request` | URL is rewritten to `api.github.com/repos/.../readme` |
| `test_github_readme_request_has_accept_header` | `Accept: application/vnd.github.v3+json` is set |
| `test_github_readme_tree_branch_request` | `/tree/{branch}` variant rewrites correctly (branch discarded) |

**Spot-check the API endpoint directly:**

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/lycheeverse/lychee/readme
# Expected: 200
```

---

## Progress Log

| Phase | Status | Notes |
|-------|--------|-------|
| Phase I — Issue Selection | ✅ Complete | Issue #2128 selected, fork created, comment posted, maintainer confirmed |
| Phase II — Reproduce & Plan | ✅ Complete | Reproduction steps documented, solution plan finalized with maintainer directives |
| Phase III — Implementation | ✅ Complete | `quirks/mod.rs` updated, tests added, branch pushed, PR submitted to upstream |
| Phase IV — PR Submission | ✅ Complete | [PR #2239](https://github.com/lycheeverse/lychee/pull/2239) submitted and actively reviewed by @katrinafyi and @mre |

---

## Phase IV — Review Feedback & Changes

PR #2239 received two rounds of detailed review from @katrinafyi (Member) and one review from @mre (owner). Below is a summary of the key feedback items and how each was addressed.

### Round 1 — @katrinafyi

| Feedback | Resolution |
|----------|-----------|
| Do not inject the GitHub token inside `Quirks` — add a dedicated chain handler in `website.rs` instead | Removed token injection from `Quirks`. Added `GitHubTokenHandler` implementing `Handler<Request, Status>` in `checker/website.rs`. It fires after Quirks has rewritten the URL to `api.github.com` and injects `Authorization: Bearer` only for requests targeting that host. |
| Use `captures.expand()` for URL construction instead of `format!()` | Rewrote the Quirk's rewrite closure to use `captures.expand("https://api.github.com/repos/$owner/$repo/readme", &mut api_url)` for the base URL, then `Url::query_pairs_mut().append_pair("ref", branch.as_str())` for the optional branch parameter. This ensures correct percent-encoding of branch names. |
| Fold the four separate README tests into a single rstest mapping input URL to expected output URL | Replaced the four individual test functions with one `#[rstest]` parameterized test (`test_github_readme_quirk`) covering base repo, branch, and non-matching cases. |
| `GitHubTokenHandler` should hold `Option<String>` to keep the chain statically defined | Changed `token: String` to `token: Option<String>`. The default chain is now always `[Quirks, GitHubTokenHandler, RemapChainHandler, credentials, WebsiteChecker]` with no conditional pushes. |
| Remove the mundane `GitHubTokenHandler` unit tests | Removed both `test_github_token_handler_injects_bearer_for_api_host` and `test_github_token_handler_skips_non_api_hosts`. |
| CLI integration tests must exercise the fragment quirk logic and verify the `Authorization` header | The original tests used `--remap api.github.com [mock]` with a `github.com#readme` input, but `--remap` fires on the original URI in `Client::check()` before the chain runs, so the mock never received the Quirks-rewritten URL. Fixed by adding `RemapChainHandler` to the chain (after `GitHubTokenHandler`) so remaps are also applied post-Quirks. The original test structure was restored: wiremock asserts on the `Authorization: Bearer` and `Accept` headers. |
| Branch capture group `[^/]+` does not allow `/` in branch names | Opened a design discussion (see thread). `[^#]+` was initially tried but is incorrect because it conflates path segments with the branch name. Reverted to `[^/]+` — URLs with slash-containing branch names fall through to a normal HTTP check, which is the safe and honest behavior given the ambiguity. |

### Round 2 — @mre (owner)

@mre reviewed the changes, indicated they would not mind merging, and asked for clarification on why `GitHubTokenHandler` and `RemapChainHandler` were needed. A response has been drafted and posted on the PR thread.

---

## Final Architecture

The PR touches four files:

| File | Change |
|------|--------|
| `lychee-lib/src/quirks/mod.rs` | New `GITHUB_README_PATTERN` regex and `Quirk` that rewrites `github.com/owner/repo#readme` to the GitHub Readme API endpoint using `captures.expand()` and `query_pairs_mut()` |
| `lychee-lib/src/checker/website.rs` | `GitHubTokenHandler` (Bearer injection post-Quirks) and `RemapChainHandler` (applies remaps after Quirks for integration test interception); static chain definition |
| `lychee-lib/src/client.rs` | Passes `remaps` to `WebsiteChecker::new()` so `RemapChainHandler` can access them |
| `lychee-bin/tests/cli.rs` | Integration tests using wiremock that assert on `Authorization: Bearer` and `Accept: application/vnd.github.v3+json` headers end-to-end |

---

## Links

- [lycheeverse/lychee GitHub repo](https://github.com/lycheeverse/lychee)
- [Issue #2128](https://github.com/lycheeverse/lychee/issues/2128)
- [My fork: DiazSk/lychee](https://github.com/DiazSk/lychee)
- [Working branch: fix/issue-2128-readme-api](https://github.com/DiazSk/lychee/tree/fix/issue-2128-readme-api)
- [**PR #2239** — fix(quirks): check GitHub #readme fragments via REST API](https://github.com/lycheeverse/lychee/pull/2239)
- [My comment on the issue](https://github.com/lycheeverse/lychee/issues/2128)
- [lychee CONTRIBUTING.md](https://github.com/lycheeverse/lychee/blob/master/CONTRIBUTING.md)
