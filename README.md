# URL Shortener

URL shortener powered by GitHub Pages + GitHub Actions. No servers, no DB to run.

## Overview
- Branch strategy: all automated writes land in the `pages` branch (also used by GitHub Pages).
- Data source of truth: `data/links.json` (client-side redirects via `404.html`).
- UI (`index.html`): status-only page that lists links and provides issue shortcuts:
  - New random shortlink issue (body prefilled with `URL:`)
  - Request reserved slug (issue template, fill `URL:` + `SLUG:`)
  - Request edit (issue template, fill `EDIT:` + `URL:`)
  - Request delete (issue template, fill `DELETE:`)
- Approval: reserved/edit/delete requests require a maintainer to comment "Approved" (case-insensitive) or close the issue as "completed". Only users with `admin`, `write`, or `maintain` are accepted. Requests opened by a maintainer are auto-approved.
- Metadata: `data/repo.json` and `data/domain.json` are auto-populated by workflows.

## Setup
1) DNS: point your custom domain (e.g., `short.example.com`) to `USERNAME.github.io.` via CNAME.
2) Pages: set repository Settings → Pages → Branch to `pages` (root).
3) Custom domain (optional):
   - Add `CNAME` with your domain, or
   - Run the workflow with `custom_domain` input. The workflow will write both `CNAME` and `data/domain.json`.

## Create links (via Issues)
### Random (auto-approved)
- Open the site (e.g., `https://short.example.com`) and click "New random shortlink issue", or open a new issue yourself.
- In the issue body, include exactly one field:
  ```text
  URL: https://example.com/path
  ```
- The workflow treats an issue as "random" if it has a `URL:` line and no `SLUG:` line.
- It generates a Base62 slug (6 chars, excluding easily-confused chars), appends the entry to `data/links.json`, commits to `pages`, and comments back the result as a clickable short URL:
  - `Created shortlink: https://your.domain/slug → https://example.com/path`
- The issue is then closed.

### Reserved (requires approval)
- Click "Request reserved slug" (issue template), and include two fields:
  ```text
  URL: https://example.com/path
  SLUG: your-slug
  ```
- The workflow detects `SLUG:` and switches to reserved mode.
- On open it validates immediately: invalid slug format, banned/invalid URL, or an already-taken slug gets an explanatory comment and the issue is closed as "not planned".
- If the requester is a maintainer, the link is created right away (auto-approved).
- Otherwise it posts an acknowledgement comment and waits. A maintainer (permission: `admin`/`write`/`maintain`) then either:
  - comments `Approved`, or
  - closes the issue as **"Close as completed"**.
- To reject, close the issue as **"Close as not planned"**.
- The workflow verifies permission, creates the link, comments back the clickable short URL, and closes the issue.

## Edit links (via Issues)
- Click "Request edit" (issue template), and include two fields:
  ```text
  EDIT: your-slug
  URL: https://example.com/new-path
  ```
- On open it validates: the slug must exist and the new URL must be valid and not banned.
- Maintainer requests are auto-approved; others need a maintainer `Approved` comment or "Close as completed".
- On approval the link's target URL is updated (`updatedAt`/`updatedBy` recorded) and the result is commented back.

## Delete links (via Issues)
- Click "Request delete" (issue template), and include one field:
  ```text
  DELETE: your-slug
  ```
- On open it validates that the slug exists.
- Maintainer requests are auto-approved; others need a maintainer `Approved` comment or "Close as completed".
- On approval the link is removed from `data/links.json`.
- Deleting the original GitHub issue that created a link also removes that link (maintainers only, via GitHub's issue delete).

## Lists and ordering
- The UI renders two sections from `data/links.json`:
  - "Reserved (pinned)": entries with `reserved: true` (highlighted), always shown first
  - "Automatic (latest first)": all other entries, sorted by `createdAt` descending
- Each item includes a copy button for the short URL.

## Redirects
- `404.html` loads `data/links.json` with cache-busting (`?ts=`) and performs client-side redirects based on `location.pathname`.

## Workflows
- File: `.github/workflows/shortlinks.yml`
- Triggers
  - `push` on `main` or `master`: write repo/domain metadata to `pages`
  - `issues.opened`: handle requests (random / reserved / edit / delete)
  - `issue_comment.created`: on "Approved" from an authorized user, apply the pending request
  - `issues.closed`: "Close as completed" approves (authorized users only — others are reopened), "Close as not planned" rejects
  - `issues.deleted`: remove any link whose `issueNumber` matches the deleted issue
  - `workflow_dispatch`: optional manual run for maintenance
- Git operations: all commits target `pages`. Pushes use fetch + rebase + retry to avoid non-fast-forward errors.
- Node execution: Node code is executed via heredoc under CommonJS and wrapped in an async IIFE for compatibility.

## Data model
- `data/links.json`
```json
{
  "version": 1,
  "links": [
    {
      "slug": "kv",
      "url": "https://example.com",
      "reserved": true,
      "createdAt": "2025-01-01T00:00:00.000Z",
      "createdBy": "github-actions[bot]",
      "issueNumber": 123
    }
  ]
}
```
- `data/banned.json`
```json
{
  "version": 1,
  "urls": [],
  "hosts": []
}
```
- `data/repo.json` auto-populated as `{ "repo": "OWNER/REPO" }`.
- `data/domain.json` populated from `CNAME` or workflow input.

## Validation & guardrails
- Banned URLs/hosts: defined in `data/banned.json`; both UI (pre-check) and workflows enforce it.
- Duplicate destination URL (random links): instead of creating a duplicate, the existing short link is returned.
- Already-taken slug: rejected immediately when the request is opened (with the current target shown).
- Slug format: `^[A-Za-z0-9-_]+$`.
- Permission check for approvals: only `admin`, `write`, or `maintain` can approve (comment or close-as-completed). Unauthorized "Close as completed" attempts are reopened automatically.
- Short-URL domain in comments: resolved from `data/domain.json`, falling back to `CNAME`, then `OWNER.github.io/REPO`.

## Troubleshooting
- New link not visible: wait for Pages redeploy (typically 30–90s) and hard refresh.
- UI shows wrong domain: ensure `CNAME` is correct; workflow will mirror it into `data/domain.json`.
- Issue didn’t create a link:
  - Random: verify the body has only `URL:` and no `SLUG:` line.
  - Reserved: verify both `URL:` and `SLUG:` exist, then an authorized maintainer must comment `Approved`.
- Repo not detected in UI: ensure `data/repo.json` exists on the `pages` branch (the workflow writes it automatically on push and during create steps).

## Limitations
- Redirects are client-side (no server 301/302).
- Anonymous GitHub API search used by the UI may be rate-limited.

## License
- Use and adapt freely for your project needs.


