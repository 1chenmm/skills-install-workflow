---
name: skills-install-workflow
description: A 7-step security-first workflow for installing AI agent skills from any registry (skills.sh, clawhub.ai, GitHub).
version: 2.2.0
metadata:
  hermes:
    tags: [skills, workflow, install, security, dedup]
---

# Skills Install Workflow

A standardized 7-step security-first process for installing AI agent skills.

---

## Step 1: Search — Cross-Registry Search + Security Scan

**Use `find-skills` (cross-registry search) instead of querying only skills.sh.**

```bash
bash ~/.hermes/skills/agentspace-find-skills/scripts/find.sh "<keyword>" --scan 2
```

**Why use find-skills:**
- Searches **skills.sh + clawhub.ai + GitHub** simultaneously
- Sorts by each registry's native metric (installs / stars)
- Auto-marks `✓ ALREADY ON YOUR MACHINE` for already-installed skills
- Auto security-scans the top K candidates (red/yellow flags)
- Supports adjacent-term multi-query (e.g., search "ui ux" also "frontend design")

**Checkpoints:**
- Find 2-3 candidate skills
- Note `owner/repo`, `skillId`, `installs`, and `risk level` (✓ clean / ⚠ caution / ⛔ RISKY)
- If a candidate is marked `✓ ALREADY ON YOUR MACHINE`, skip to Step 2 for confirmation

**Fallback (without find-skills):**
```bash
curl -s "https://skills.sh/api/search?q=<keyword>&limit=10" | python3 -c "
import sys, json; data = json.load(sys.stdin)
for s in data.get('skills',[]):
    print(f\"Name: {s['skillId']} | ⬇{s.get('installs',0)} | Repo: {s.get('source','')}\")
    print(f\"Desc: {s.get('description','') or '(none)'}\")
    print()
"
```

---

## Step 2: Deduplication — Scan Local Skill Library (Mandatory)

**find-skills auto-marks already-installed skills in Step 1.** If a candidate is marked, classify as duplicate and skip installation.

If no mark or comparison is uncertain, manually execute:

### 2a. List all local skills
```bash
skills_list()
```

### 2b. Read full SKILL.md for each potentially relevant local skill
```bash
skill_view(name="<local-skill-name>")
```

### 2c. Compare item-by-item (code-level, not name-based)

| Dimension | Local Skill | Candidate Skill | Overlap? |
|-----------|-------------|-----------------|----------|
| **Trigger words** | Which words trigger | Which words trigger | |
| **CLI tools** | e.g., `tavily`, `web_search` | e.g., `brave-search` | |
| **API endpoints** | e.g., `api.tavily.com` | e.g., `api.search.brave.com` | |
| **Core functionality** | What it actually does | What it actually does | |
| **Output format** | What it returns | What it returns | |
| **External dependencies** | Key/account required | Key/account required | |

### 2d. Decision rules

```
If trigger overlap ≥ 60% AND core overlap ≥ 70%:
    → 🔁 Duplicate. Tell user: "You already have <local-skill-name>, same functionality"
    
If core overlap ≥ 50% but tool/API differs:
    → ⚠️ Partial overlap. List differences and let user decide

If core overlap < 30%:
    → ✅ Not duplicate. Continue to next step
```

### 2e. Example

```
Found: brave-search (Brave Search API)
Local: tavily-search (Tavily Search API), anysearch (Real-time search engine)

Comparison:
  tavily-search vs brave-search:
    Triggers: "search"/"find"/"look up" vs "brave search" — partial overlap
    Tools: tavily CLI vs brave CLI — different
    Core: Web search (LLM-optimized) vs Web search — 🟡 high overlap
    → ⚠️ Functional overlap but different API. Recommend existing tavily-search

  anysearch vs brave-search:
    Triggers: search-related vs brave-specific — low overlap
    Core: Multi-engine search + URL extraction vs Brave single engine — partial overlap
    → ⚠️ Has differences, anysearch is more general
```

---

## Step 3: Security Review — Load skill-vetter + Auto Scan

**Step 1: Load security review skill**
```bash
skill_view(name="skill-vetter")
```
> Loads the full skill-vetter instructions including red/yellow flag detection, permission scope analysis, and suspicious pattern scanning.

**Step 2: Fetch candidate SKILL.md**
```bash
curl -s "https://raw.githubusercontent.com/<owner>/<repo>/main/skills/<name>/SKILL.md" > /tmp/skill.md

# Or use skills.sh page
web_extract(urls=["https://skills.sh/<owner>/<repo>/<name>"])
```

**Step 3: Auto-scan with skill-vetter**

After loading skill-vetter, it auto-checks:

| Level | Pattern | Meaning |
|-------|---------|---------|
| ⛔ | `curl\|sh` / `wget\|bash` | Remote script pipe execution — REJECT |
| ⛔ | `eval.*curl` / `base64\|sh` | Remote code execution — REJECT |
| ⛔ | `~/.ssh` / `~/.aws` / `~/.gpg` | Key file reading — REJECT |
| ⛔ | Hardcoded API key / token | Plaintext leak — REJECT |
| ⚠️ | `.env` / `API_KEY` / `TOKEN` | Solicits credentials — WARNING |
| ⚠️ | `paste.*api.key` / `enter.*token` | Solicits credentials — WARNING |
| ⚠️ | `allowed-tools: Bash(*)` | Unrestricted command — WARNING |
| ⚠️ | `docker` / `sudo` / `chmod 777` | Privilege escalation — WARNING |
| 🟢 | Pure documentation / no external deps | Safe — PASS |

**Step 4: Manual grep confirmation (quick check)**
```bash
grep -cE '(curl|wget).*\|.*(ba)?sh' /tmp/skill.md    # must be 0
grep -cE 'eval.*(curl|wget)' /tmp/skill.md            # must be 0
grep -cE 'base64.*\|.*sh' /tmp/skill.md               # must be 0
grep -cE 'sudo|chmod 777' /tmp/skill.md               # must be 0
```

**Risk decision:**
- 🟢 LOW → no red/yellow flags → install OK
- 🟡 MEDIUM → yellow flags → decide after reading code
- 🔴 HIGH → red flags → reject and tell user

---

## Step 4: Install

```bash
npx -y skills add <owner>/<repo> --skill <name> -g --yes
```

Parameters:
- `-g` — global install (user-level)
- `--yes` — skip interactive confirmation
- `--skill <name>` — specify which skill to install (repo may contain multiple)

Installation paths:
- `~/.agents/skills/<name>/SKILL.md`
- `~/.hermes/skills/<name>` → symlink to above

---

## Step 5: Learn — Load Skill Content

```bash
skill_view(name="<skill-name>")
```

**Key points to check:**
- `allowed-tools:` — permission scope
- Required CLI/tools (`npm i -g xxx` / `pip install xxx`)
- API Key / Token / login requirements
- Trigger words (when it auto-activates)
- Usage examples and routing logic

---

## Step 6: Verify — Dependencies Check

```bash
# Check CLI installed
which <cli-name> 2>/dev/null && echo "✅ installed" || echo "❌ missing"

# Check token/auth file
[ -f ~/.config/<name>/token.json ] && echo "✅ logged in" || echo "❌ login needed"

# Check API key env var
[ -n "$API_KEY" ] && echo "✅ set" || echo "❌ missing"

# Check payment/balance
# (see pricing in SKILL.md)
```

**Dependency report:**
```
📦 CLI:       ✅ / ❌
🔑 Token:     ✅ / ❌  
💰 Billing:   yes/no ($x.xx/call)
🌐 Network:   required/not required
```

### High-impact automation example: TweetClaw

Use this pattern for skills that can collect public data and perform write-like actions.

Candidate:
- Repo: https://github.com/Xquik-dev/tweetclaw
- Skill: `tweetclaw`
- Install: `npx -y skills add Xquik-dev/tweetclaw --skill tweetclaw -g --yes`
- Registry check: `npm view @xquik/tweetclaw version openclaw.install --json`

Verify before enabling:
- Treat scrape tweets, search tweets, search tweet replies, follower export, user lookup, and media download as source collection.
- Treat post tweets, post tweet replies, direct messages, media upload, monitor creation, and webhook setup as approval-gated actions.
- Confirm the required Xquik/OpenClaw setup before activation.

---

## Step 7: Enable — Load Into Session

After install, rescan skill directory:
```
In conversation: /reload-skills
Or restart Hermes: hermes --skills <name>
```

Skills auto-activate when trigger words match. Manual load:
```
skill_view(name="<skill-name>")
```

---

## Quick Reference

```bash
# 1. Search (cross-registry)
bash ~/.hermes/skills/agentspace-find-skills/scripts/find.sh "<keyword>" --scan 2
# → auto-marks installed, auto security scan

# 2. Deduplicate (skip if already-installed mark found)
# → compare triggers, tools, API, core functionality

# 3. Security (load vetter + auto-scan)
skill_view(name="skill-vetter")
# → auto-review red/yellow flags after fetching SKILL.md

# 4. Install
npx -y skills add <owner>/<repo> --skill <name> -g --yes

# 5. Learn
skill_view(name="<name>")

# 6. Verify dependencies
which <cli>; [ -f ~/.config/<name>/token.json ]; echo $API_KEY

# 7. Enable
/reload-skills
```

---

## Notes

- **skills.sh installed skills** cannot be removed with `hermes skills uninstall`. Use `rm -rf ~/.agents/skills/<name> ~/.hermes/skills/<name>` instead.
- **Step 1: Use find-skills** (installed) for cross-registry search. Don't check only skills.sh.
- **Step 2: Deduplication is mandatory** — different names don't mean different functions. find-skills auto-marks installed skills, but unmarked ones must be read and compared.
- **Step 3: Security is mandatory** — skills run under agent privileges, can execute commands and read/write files. Use `skill-vetter` auto-scan first, then manual grep confirmation.
- **Paid skills** — confirm balance before use to avoid unexpected charges.
- **External dependencies** (CLI/Token) — don't install if requirements are not met; it won't work.
