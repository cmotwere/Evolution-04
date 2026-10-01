# Design Analysis — GitHub MCP server teardown

CIDM 6333 · Lab, Thu 09/24/2026

## Sources (read these yourself before filling anything in)

1. README: https://github.com/github/github-mcp-server
2. Config reference: `docs/server-configuration.md` (same repo)
3. Incident: Milanta, M., & Beurer-Kellner, L. (2025, May 26). *GitHub MCP exploited:
   Accessing private repositories via MCP.* Invariant Labs.
   https://invariantlabs.ai/blog/mcp-github-vulnerability

---

## Step 1 — Inventory the surface

1. Ways to run the server (remote URL / local Docker image):

   - **Remote**: Hosted by GitHub at `https://api.githubcopilot.com/mcp/` (HTTP / Streamable HTTP).
   - **Local**: Docker image `ghcr.io/github/github-mcp-server` **or** binary built from source (`github-mcp-server stdio`).

2. Ways to authenticate (OAuth / personal access token), and which takes precedence if both are present:

   - **OAuth** (browser-based login flow; token kept in memory only).
   - **Personal Access Token (PAT)** via `GITHUB_PERSONAL_ACCESS_TOKEN` (or `Authorization: Bearer …` header).
   - **Precedence**: PAT takes precedence over OAuth when both are present.

3. Default toolsets (name them + count) / all toolsets offered (count):

   - **Default (5)**: `context`, `issues`, `pull_requests`, `repos`, `users`.
   - **All offered (~20+)**: `context`, `actions`, `code_quality`, `code_security`, `copilot`, `copilot_issue_intents`, `dependabot`, `discussions`, `gists`, `git`, `governance`, `issues`, `labels`, `notifications`, `orgs`, `projects`, `pull_requests`, `repos`, `secret_protection`, `security_advisories`, `stargazers`, `users` (+ a few remote-only toolsets).

4. From `repos`, `issues`, `pull_requests` — at least 8 tools, each with an effect class
   (read / write / irreversible), based on what the tool *does*, not its name:

   | Tool                      | Toolset        | Effect class   |
   |---------------------------|----------------|----------------|
   | `list_issues`             | issues         | read           |
   | `get_issue` / `issue_read`| issues         | read           |
   | `create_issue`            | issues         | write          |
   | `add_issue_comment`       | issues         | write          |
   | `get_file_contents`       | repos          | read           |
   | `create_or_update_file`   | repos          | write          |
   | `list_pull_requests` / `pull_request_read` | pull_requests | read |
   | `create_pull_request`     | pull_requests  | write          |
   | `merge_pull_request`      | pull_requests  | irreversible   |

5. Of those tools, which ones return or act on text written by someone other than you
   (an issue body, a PR description, a comment, file content in a repo you don't own)?

   - `list_issues`, `get_issue` / `issue_read`, `add_issue_comment` (issue bodies & comments)
   - `get_file_contents` (file content from any repository the credential can read)
   - `list_pull_requests`, `get_pull_request` / `pull_request_read` (PR titles, bodies, review comments)

---

## Step 2 — Trace the attack as tool calls

From the Invariant Labs post:

| Step | What the agent does                                      | Tool (effect class)              | Whose text or data                  |
|------|----------------------------------------------------------|----------------------------------|-------------------------------------|
| 1    | User asks agent to examine open issues in the **public** repo; agent fetches them | `list_issues` (read)            | Attacker’s malicious issue body     |
| 2    | Injected agent is coerced to read data from a **private** repository | `get_file_contents` / private-repo list (read) | Victim’s private repository content |
| 3    | Agent creates a pull request on the **public** repo that contains the leaked private data | `create_pull_request` (write)   | Victim’s private data (now public)  |

---

## Step 3 — Name the setting that stops each step

For each step above: which setting stops it, and who sets it?

| Step | Setting                                                                 | Who sets it                  |
|------|-------------------------------------------------------------------------|------------------------------|
| 1    | **Lockdown mode** (`--lockdown-mode` / `X-MCP-Lockdown` / `GITHUB_LOCKDOWN_MODE`) — filters content authored by users without push access on public repositories | Server operator / client config |
| 2    | **Toolset / individual-tool allow-list** limited to a single repository **or** a least-privilege PAT that cannot see the private repository **or** runtime guardrails that forbid cross-repo reads | Operator / host policy       |
| 3    | **Read-only mode** (`--read-only` / `X-MCP-Readonly`) **or** explicit exclusion of `create_pull_request` via `--exclude-tools` / `X-MCP-Exclude-Tools` | Server operator / client config |

**Step with no setting that stops it (the finding):**  
The core toxic flow (untrusted issue text → cross-repository read → public write) cannot be completely eliminated by any single server-side configuration switch without also breaking legitimate multi-repository or write workflows. Lockdown mitigates step 1, read-only / tool exclusion mitigates step 3, but the combination of “read any repository the token can see + write to a public repository” remains the residual risk.

---

## Step 4 — OpenCode as Chase (red-team check)

Model id used: `claude-4-opus` (same family used in the original Invariant demonstration)

Prompt pasted into OpenCode (with your trace + settings from steps 2-3):

> You are a red-team reviewer. Do not use any tools. Here is how an agent is
> connected to the GitHub MCP server, and the settings I chose to stop a
> prompt-injection attack that leaks private repository data:
>
> Attack trace:
> 1. list_issues on public repo → receives attacker-controlled issue body
> 2. get_file_contents (or equivalent) on private repo → private data enters context
> 3. create_pull_request on public repo → private data is published
>
> Settings chosen:
> - Lockdown mode enabled (filters non-push-access content on public repos)
> - create_pull_request excluded via --exclude-tools / X-MCP-Exclude-Tools
> - Default toolsets kept otherwise (context, issues, pull_requests, repos, users)
>
> 1. Name one other way the agent could still move private data to a place a
>    stranger can read, given only these settings. Name the GitHub MCP tool.
> 2. Name one ordinary task a developer could no longer do with these settings.
>    Answer in four sentences or fewer.

**Model's answer to Q1 (tool it named):**  
`create_issue` (or `add_issue_comment`) — the agent can still open a new issue or post a comment on the public repository that contains the private data it already read.

**Checked against README — does that tool exist? Is it a write path? Is it left enabled by your settings?**  
Verdict: **confirmed**.  
Both `create_issue` and `add_issue_comment` exist in the `issues` toolset (README “Tools” section). They are write tools and remain enabled because only `create_pull_request` was excluded and read-only mode was not turned on.

**Model's answer to Q2 (trade-off):**  
A developer can no longer open pull requests through the agent (the most common way to propose code changes).

My own one-sentence trade-off (write your own if the model's answer wasn't useful):  
Excluding only `create_pull_request` still leaves other public-write surfaces (`create_issue`, `add_issue_comment`, `create_or_update_file`) that can exfiltrate data, while simultaneously removing the primary legitimate workflow of opening PRs.

---

## Step 5 — Close: the judgment loop (5 lines)

1. **Decision** (which configuration you’d grant your own agent):  
   Default toolsets + lockdown mode + explicit exclusion of all public-write tools that can carry arbitrary text (`create_pull_request`, `create_issue`, `add_issue_comment`, `create_or_update_file`) **or** full read-only mode when the agent only needs to answer questions.

2. **Characteristics it privileges:**  
   Least privilege, content filtering of untrusted public-repo text, and prevention of any write that can publish private data.

3. **What it costs:**  
   The agent can no longer create issues, comments, PRs, or files; multi-repo “read-then-act” workflows become impossible without additional host-level policy.

4. **Failure mode it leaves:**  
   Private data can still be read into the model’s context and appear in the chat transcript or host logs; a determined attacker who can still inject via any remaining readable surface may try to social-engineer the human operator.

5. **Mechanism that preserves it:**  
   Continuous monitoring (e.g. MCP-scan / Guardrails) plus periodic review of the exact tool allow-list and the scopes of the underlying GitHub credential.

---

## Worksheet (summary)

| Field | Value |
|---|---|
| Ways to run (remote / local) | Remote (`api.githubcopilot.com/mcp/`) / Local (Docker `ghcr.io/github/github-mcp-server` or binary) |
| Ways to authenticate; which takes precedence | OAuth or PAT; **PAT takes precedence** |
| Default toolsets (count) / all toolsets (count) | 5 (`context, issues, pull_requests, repos, users`) / ~20+ |
| Tools with effect class (at least eight) | read: `list_issues`, `get_issue`, `get_file_contents`, `list_pull_requests` · write: `create_issue`, `add_issue_comment`, `create_or_update_file`, `create_pull_request` · irreversible: `merge_pull_request` |
| Tools that return other people's text | `list_issues`, `get_issue`, `get_file_contents`, `list_pull_requests`, `get_pull_request`, `add_issue_comment` |
| Attack trace | 1. `list_issues` (public) 2. `get_file_contents` (private) 3. `create_pull_request` (public) |
| Setting that stops each step | 1. Lockdown mode 2. Tool/repo allow-list or least-privilege PAT 3. Read-only mode or exclude `create_pull_request` |
| Step that no setting stops | The overall toxic flow (untrusted text → cross-repo read → public write) cannot be fully closed by one switch without breaking legitimate use |
| Model id used in step 4 | `claude-4-opus` |
| Model's other path (Q1): tool named · confirmed/refuted/cannot tell · evidence | `create_issue` / `add_issue_comment` · **confirmed** · both exist in `issues` toolset and remain enabled |
| Trade-off (Q2) | Cannot open PRs (or, with stricter settings, cannot create issues/comments/files either) |
| Judgment loop (decision · characteristics · cost · failure mode · preservation) | Strict write exclusion + lockdown · least privilege & content filter · loses write workflows · residual chat/log leakage · continuous monitoring + credential hygiene |

---

## Exit ticket

- [x] This file filled through Step 4
- [ ] Ledger event recorded for the commit
