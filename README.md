<div align="center">
  <img src="docs/icon.png" alt="AI Security" width="96" height="96" />
  <h1>AI Security</h1>
  <p><em>Automated OSS security auditing that turns merged fixes into a live, provable PR record</em></p>
  <p><a href="https://github.com/bunlongheng/ai-security">Repo</a> &middot; <a href="https://bunlongheng.com/projects?name=ai-security">Portfolio</a></p>
  <img src="docs/social-preview.png" alt="AI Security - preview" width="820" />
</div>

---

# ai-security

Turns real, merged security fixes to popular open-source projects into a public, provable engineering record - discovered, audited, and tracked automatically.

![PR board](docs/screenshots/board.png)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?logo=typescript)
![Vitest](https://img.shields.io/badge/tests-vitest-6e9f18?logo=vitest)
![SQLite](https://img.shields.io/badge/data-SQLite-003b57?logo=sqlite)

## Contents

- [Why this exists](#why-this-exists)
- [Features](#features)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Project layout](#project-layout)
- [License](#license)

## Why this exists

Anyone can claim they "contribute to open source." A merged PR fixing a real vulnerability in a 27,000-star repo is proof a stranger can verify in thirty seconds - no resume bullet gets that kind of credibility for free.

- **Third-party-verified proof of skill.** Every row on the board links to a real, live GitHub PR. Nobody has to take your word for it.
- **A defensible audit trail, not a vibes-based fix.** Each PR is backed by a stored record of every file checked, every competing finding considered, and why the winning one beat the runner-up - see the [audit trail](#features) below.
- **Interview-ready by construction.** Because the reasoning behind each fix is captured at audit time, you can walk an interviewer through the finding, the fix, and the trade-offs months later without re-deriving it from a diff.
- **Signal over spam.** A one-PR-per-repo rule plus confidence scoring means maintainers see one well-reasoned fix, not a drive-by flood - the thing that gets PRs merged instead of closed.
- **The tedious part is automated.** An hourly scan does the searching across dozens of repos so the only manual work left is auditing a shortlisted candidate and reviewing the generated PR.

## Features

- **Hourly discovery scan** - `scripts/discover.mjs` scans a curated seed list plus live GitHub search, scores each candidate issue by repo vitality, issue age, maintainer engagement, and a security bonus, and filters out anything below a confidence threshold
- **Full audit trail per PR** - each submission stores which files were audited and why, a tournament of every candidate finding considered (ranked by realness, patchability, and mergability, with eliminated candidates and their reasons kept), the winning fix, and generated interview talking points
- **Live PR board** - every submitted PR grouped by real-time status (Open / Merged / Draft / Rejected), pulled straight from the GitHub API and cached for 5 minutes
- **Category and repo-type tagging** - PRs are tagged security / correctness / performance / feature / enhancement, and repos are tagged by kind (library, tool, AI, app, API, game) for fast scanning of the board
- **Tech-stack detection per repo** - languages and key frameworks shown per audited repo, sourced from the GitHub API

## Architecture

Two independent producers feed one SQLite-backed catalog, which a Next.js app reads and merges with live GitHub state at request time.

```mermaid
flowchart LR
    subgraph Discovery
        CRON["scripts/discover.mjs\n(hourly cron)"] -->|confidence score| PIPE[("web/data/pipeline.json")]
    end
    subgraph Audit
        SKILL["/security-audit skill"] -->|"POST /api/audit"| API[app/api/audit]
        API --> DB[("db/audit.db\naudit_reports")]
    end
    subgraph Catalog
        CAT["web/lib/prs.ts\nPR_CATALOG"]
    end
    GH[("GitHub REST API")]
    CAT --> MERGE["fetchLivePRs()"]
    GH --> MERGE
    MERGE --> BOARD["/prs board"]
    DB --> DETAIL["/prs/:owner/:repo/:number\naudit detail"]
    BOARD --> DETAIL
```

| Layer | Role |
|-------|------|
| `scripts/discover.mjs` | Finds and scores candidate repos/issues, independent of the web app |
| `/security-audit` skill | Performs the actual audit on a chosen repo and posts the report |
| `db/audit.db` (better-sqlite3) | Stores one audit report per `(repo, pr_number)`: files audited, findings tournament, PR body, talking points, tech stack |
| `web/lib/prs.ts` | Hand-curated catalog of submitted PRs, merged with live GitHub status |
| `/prs` | The public board, grouped by status |
| `/prs/[owner]/[repo]/[number]` | Per-PR detail page rendering the stored audit report |

## Tech stack

| Layer | Tech |
|-------|------|
| Framework | Next.js 16 App Router (server components) |
| Language | TypeScript 5 (strict) |
| Styling | Tailwind CSS |
| Data | better-sqlite3 (audit reports), GitHub REST API v3 (live PR/issue state) |
| Discovery | Node.js + `gh` CLI |
| Tests | Vitest |

## Quick start

```bash
# 1. Clone
git clone https://github.com/bunlongheng/ai-security.git
cd ai-security/web

# 2. Install dependencies
npm install

# 3. (Optional) add a GitHub token to raise the API rate limit from 60 to 5,000 req/hr
echo "GITHUB_TOKEN=ghp_..." >> .env.local

# 4. Run the dev server
npm run dev        # http://localhost:3018
```

Visiting `/` redirects to `/prs`, the live board.

### Running the discovery scan

```bash
# From the repo root
node scripts/discover.mjs
```

Scans every repo in `SEED_REPOS` plus live GitHub search results, and keeps a candidate only if:
1. The repo has real vitality: at least 1 merged PR in the last 60 days
2. It has an open bug issue with no competing pull request
3. Its confidence score is 30 or higher

Confidence is a 0-100 score:

| Signal | Points |
|--------|--------|
| No competing PR exists | +40 |
| Repo vitality >= 8 merged PRs/60 days | +25 |
| Repo vitality >= 5 merged PRs/60 days | +18 |
| Repo vitality >= 3 merged PRs/60 days | +12 |
| Repo vitality >= 1 merged PR/60 days | +5 |
| Issue age 30-180 days | +20 |
| Issue age 181-365 days | +10 |
| Issue has a maintainer comment | +15 |
| Flagged as security-relevant | +20 |

Results are written to `web/data/pipeline.json` (gitignored - runtime data). Schedule it hourly with cron or any task scheduler:

```cron
13 * * * * cd /path/to/ai-security && node scripts/discover.mjs >> /tmp/discover.log 2>&1
```

### Auditing a candidate and recording the trail

Run the `/security-audit <owner/repo>` skill against a shortlisted candidate. It performs the audit and posts the result to `POST /api/audit`, which stores it in `db/audit.db` keyed by `(repo, pr_number)`. The detail page at `/prs/:owner/:repo/:number` renders it automatically once a matching PR is in the catalog.

![Audit trail detail page](docs/screenshots/audit-detail.png)

### Adding a submitted PR

Edit `web/lib/prs.ts` and append to `PR_CATALOG`:

```ts
{
  number: 10465,       // actual PR number on GitHub
  repo: "owner/repo",  // e.g. "marimo-team/marimo"
  title: "fix: short description",
  issue: "#9624",
  submittedAt: "2026-08-05",
}
```

The board picks up live status on the next page load (cached for 5 minutes).

## Configuration

| Env var | Default | Purpose |
|---------|---------|---------|
| `GITHUB_TOKEN` | none (60 req/hr, unauthenticated) | Raises the GitHub API rate limit to 5,000 req/hr; needs `read:repo` scope |

## Project layout

```
ai-security/
- scripts/
  - discover.mjs          # hourly discovery engine (confidence scoring, seed repos)
- db/
  - migrations/           # SQLite schema for audit_reports
- web/
  - app/
    - prs/
      - page.tsx                          # the live PR board
      - [owner]/[repo]/[number]/page.tsx  # per-PR audit detail page
    - api/
      - audit/         # POST: store an audit report
      - opportunity/   # PATCH: favorite / skip a discovered candidate
      - scan/          # POST: trigger a discovery scan on-demand
  - lib/
    - prs.ts            # PR_CATALOG + live GitHub status merge
    - db.ts             # audit_reports read/write (better-sqlite3)
  - data/               # gitignored runtime data (pipeline.json written by discover.mjs)
```

## License

[MIT](LICENSE) (c) Bunlong Heng

---

<p align="center">
  <sub>Built by <a href="https://bunlongheng.com">Bunlong Heng</a> &middot; <a href="https://bunlongheng.com/projects/ai-security">See it in my portfolio &rarr;</a></sub>
</p>

---

<div align="center">

<a href="https://bunlongheng.com"><img src="https://img.shields.io/badge/-bunlongheng.com-3A3A3C?style=for-the-badge&amp;labelColor=2A2A2C&amp;logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABwAAAAcCAMAAABF0y%2BmAAADAFBMVEXx8vLq6v%2F19fX09PT8%2Ff309PT5%2BfnAwMBMaXHx8vL19fX5%2Bfny8vL09PT39%2Ff6%2Bfn6%2Bvr09PTx8fH4%2BPn%2F8vLy8%2FP09PTz8%2FPy8vL%2F%2F%2F%2Fz9PTz9PP19fby8vP19PXz8%2FT19%2Fb08%2FTx8vLy8vL09vby8%2FL29vf19fTv8PDz9vb7%2Bvn%2F%2Ff%2F9%2Ff3w8vH29fb%2F%2FP339%2Ff5%2Bfn59%2Ff19PT09PR7rao3VF3%2F%2Fv8AKDR6sqsBV1vv8fL4%2BPiux8b8%2Bvq90NDy8%2FR%2BlpppnZyZqrACLTxclZJSkI9en5ssZ2luqaUAeWgqbW4BX1gCfG0%2Fa3QaVG0MN0UKNUKUpKsGpYMAKjoRQ1UDf3AQRloJqIgEm32d0cUBg3ElbnQIln8eVWgrmIdP0KzF5d5s2acdfHwppJCP5MMXl4Tz8%2FLv8%2FP39%2FiewL5%2BqahYf4IrTFZkk5KlxsREiIYAHiqDnKFmnJowc3IdVllvpKK7zc3D2tnk5%2BmuycdwpKKuub2yv8Ly%2B%2FkAMT7z%2Bvjq7e09gH93q6kGdWpGgoJZmJSlw8M7c3WwyceTw76w1M8mYmpFfX8lW2AFOEpGiYhxqqRrqaRb2rdIsZ8AVE91jpU7d3t3saqBoKhako4AWVEcUlg3aWwDupAHb2YwmIkPP1ERSVA7WmZ8tq8HT14oZW%2BYuLsMl3wQT10ROksENUc8Z3MJSGVLh4dqpJ%2BXr7U51awUgHM4v6ad1MkPuJZAiowVd3IEhnZH0aoYrI05p5sDtYwkzZ8EPViYxsRel5USbHAIgmwIM0gQamzU4uF2malQi4oWQFQ1zaYrp4UXlYwfiHoyxqFE1q4qeX4yn5JIxKkdrX1b1rAbWGvd5Oh91sRijpomsJEVc3Ukc34dqIcwuJVKwZIXkoQZcXlNv5g%2BqZUZvpNl2rPG9N0WkIoNqouW4sVPwp%2BM0b6h0ssfq4d%2F1r8hpo%2Fh9e4AamUjnI9avaBixqgklYWSwL4xtItUxplYl5qZ58mq78wMgnYXfnlOmJaR1L%2Bj78ny9fS%2FrXQPAAAAFXRSTlP7Brvx%2FsJhAgD87r4U72C4uGH8vhRDodYHAAAACXBIWXMAAAsTAAALEwEAmpwYAAAC6UlEQVQokS2Sd3BUVRSHb0KS3QRCiZw5t%2By9j33zXjZkyUs2uekhEAi9dwRCL4JUEQRFAQUs9A4WOopKU0GpFoqCBZUivaqoFKkWLIS5GX7%2FfnPO78yZj%2Fj8UTViK6HggvNgSGuluEZRKbZqlN9H%2FFWiWUzIFQK5fhjOIRTDoqv4ia86S4q3KSByHdSqIhxofBKr5iNxLMm2bQqWVkorJ%2BAkO46nkQJhceSRmCTDvKJIJBLJzMxMLU6POJ4eBPF1KpNEbdt2Cms4LCcnZ%2FX6D3d%2BsKJJ7VSlNLoqgbjUtmkKKxlVmpf3bsFXW3d1b77gueJAQIPQBCilkMKemNi23Ts9T35%2FaMe2rsvmTypylGUFCSIYWJLXrm2nUwOPfNO5w%2B73Okxnjqe0JigQEL3JizoeONr%2F2u99vju4fd1rdZnyspQmnGvAUBprWNqz348%2FhXN%2F6P1%2By4Uvs%2FJkpRQRwpJBRVn7vf16Dfitz5XevQqWt3yalXtKaRIUQmglWfsu5y7%2BMrDs0TMftWo1ZVy64xioOVpcIdt8%2Fvqvf9%2F859KJsrLmrzA%2BOKC0IkFEtJza7JOfr%2FYd0P9G98tdOxe0Hp9qJrOIBgDuOeEtp%2F%2B4d%2FvWX%2F%2F9u%2B%2FtlaUTmOWZg7SQEEyuFe52%2BML9O3cZO%2F5xl06zn2VcadOJUqJTzt7a82XfP7sx9vmGNm1m1MstV4IHCVAAoTLqL1372ddn%2Fz%2B2v%2FWq12c1fSpsaeQWAZAANntxzuI3e3zbo8XGpm%2FMnNqiXrh%2BljaPd6UUabkjps19tdGnX2zKb9b4pTFDCzOMLmYtpW5h%2BpCRTeYtyV%2BT36zx808OL4xYGoFrkqDTbJpSXLdRgwbPZL%2BQnT129OOPFVEEsAMJpHKdeKMJgJSAiALdjJANQKnRJI4R6boAKEwLCiGoBBegQjBftQo1KQWOlplFKSV9qGaF1NpAE7QQESTlMSy6pp%2F4%2FFFVYxMNRBTaiM0tkImxNaL8vgfDR8gvoYRaxgAAAABJRU5ErkJggg%3D%3D" alt="bunlongheng.com"></a>
<a href="https://www.linkedin.com/in/bunlongheng/"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"></a>
<a href="https://www.instagram.com/ibunlong/"><img src="https://img.shields.io/badge/Instagram-C13584?style=for-the-badge&logo=instagram&logoColor=white" alt="Instagram"></a>
<a href="mailto:bheng.code@gmail.com"><img src="https://img.shields.io/badge/Email-2E7D32?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"></a>

<br>

Built by **[Bunlong](https://bunlongheng.com)** &nbsp;&middot;&nbsp; [more apps](https://bunlongheng.com/projects)

</div>
