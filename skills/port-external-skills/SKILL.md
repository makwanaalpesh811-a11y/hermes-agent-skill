---
name: port-external-skills
description: Port Claude Code / OpenClaw / Codex skill repos into Hermes.
version: 1.0.0
author: SenseNova (curator)
license: MIT
platforms: [android, linux, macos, windows]
metadata:
  hermes:
    tags: [skills, porting, claude-code, openclaw, codex, social-media]
    related_skills: [hermes-agent-skill-authoring]
---

# Port External Skill Repos to Hermes

Many GitHub repos ship "skills" targeted at Claude Code, OpenClaw, or Codex — usually MIT-licensed, with a `skills/*/SKILL.md` layout and optional Python tools or JSON reference files. The user often wants them in Hermes because Hermes reads `SKILL.md` directly. This skill is the porting workflow: preserve the body and scripts verbatim, rewrite only the frontmatter to Hermes format.

## When to Use

- User shares a GitHub URL for a skills repo and asks "port this", "make it work here", "install in Hermes", or "clone karo".
- A skill bundle exists on disk from a previous clone and needs to become Hermes-loadable.
- Multiple skills in one repo — port as a batch, not one at a time.

Not this skill for: skills already in Hermes format (just copy the folder). Also not for skills that need a paid API to actually work — port them anyway but flag in the reply that a specific lib/client requires a paid key.

## Porting Procedure

### 1. Clone into Almari, not home

```bash
mkdir -p /sdcard/Hermes_Almari/06_TRANSIENT/<slug>_repo
cd /sdcard/Hermes_Almari/06_TRANSIENT/<slug>_repo
git clone --depth 1 <repo-url>
```

Never clone into `~` — Almari is the durable scratch space and home stays clean.

### 2. Enumerate skills and their support files

```bash
ls skills/
for d in skills/*/; do find "$d" -type f; done
```

Every skill has a `SKILL.md`. Some add `references/*.md`, `templates/`, or `scripts/*.py`. Note them — they travel with the skill.

### 3. Read frontmatter, extract body

Use `read_file` or Python `open()`. Body = everything after the closing `\n---\n`. Never rewrite the body — it is the actual instructions and any rewriting loses fidelity.

```python
def extract_body(src):
    lines = src.split("\n")
    for i, line in enumerate(lines[1:], 1):
        if line.strip() == "---":
            return "\n".join(lines[i+1:]).strip()
    return None
```

### 4. Rewrite the frontmatter to Hermes format

Hermes rules that the source repos violate:

- `description` must be **≤ 60 characters** and end with `.` — the validator ceiling is 1024 but review enforces 60.
- `author` must credit the **human contributor first**, then `Hermes Agent`.
- `related_skills` must only name skills that actually exist in `~/.hermes/skills/`. Do NOT auto-generate a self-reference or point at skills that aren't installed.
- `metadata.hermes.tags` and `metadata.hermes.related_skills` are both required.

Template:

```yaml
---
name: <skill-name>
description: <short imperative capability, ≤60 chars, ends with .>
version: 1.0.0
author: <Human Author> (<handle>), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [<topic>, <tool>, <domain>, <skill-name>]
    related_skills: [<other-installed-skills>]
---
```

For batch ports, pre-write short descriptions in a dict keyed by skill name — do not generate them from the source `description` field (those are usually 200-400 chars and are the whole reason the frontmatter has to be rewritten).

### 5. Copy to `~/.hermes/skills/<category>/<skill-name>/`

Pick the category by what the skill is: `social-media/` for Instagram/X/YouTube content, `productivity/` for planning, `research/` for analysis, `devops/` for infra. Category subdirs are created on demand.

```python
import os, shutil
skill_out = os.path.join("~/.hermes/skills/<category>", skill_name)
os.makedirs(skill_out, exist_ok=True)
# write SKILL.md
with open(f"{skill_out}/SKILL.md", "w") as f:
    f.write(new_content)
# copy all sibling files (scripts, references/, etc.)
for extra in os.listdir(src_path):
    if extra == "SKILL.md": continue
    src = os.path.join(src_path, extra)
    dst = os.path.join(skill_out, extra)
    if os.path.isdir(src):
        shutil.copytree(src, dst, dirs_exist_ok=True)
    else:
        shutil.copy2(src, dst)
```

### 6. Validate all ports in one shot

```python
import yaml
for name in skills_to_create:
    content = open(f"~/.hermes/skills/{category}/{name}/SKILL.md").read()
    assert content.startswith("---")
    fm_end = content.find("\n---", 3)
    fm = yaml.safe_load(content[3:fm_end])
    assert len(fm["description"]) <= 60
    assert fm["description"].endswith(".")
    assert "platforms" in fm
    assert "metadata" in fm and "hermes" in fm["metadata"]
```

Then smoke-test one Python tool from the batch to prove scripts survive the copy:

```bash
python3 ~/.hermes/skills/social-media/<skill>/scripts.py <test-input> | head -20
```

### 7. Verify a ported skill loads in Hermes

`skill_view(name="<skill>")` in the same session may not show the new skill (loader caches at session start). But the file is present and the next session will see it. Report this to the user honestly — don't claim a fresh `skill_view` succeeds on a just-created skill.

## Pitfalls

- **Source descriptions are 200-400 chars, Hermes caps at 60.** Extracting the first 60 chars usually produces a broken sentence or a mid-word cut. Always hand-write the short descriptions in a dict before batching — do not truncate programmatically.
- **Do not auto-derive `related_skills` from the source repo.** A ported skill listing itself as `related_skills: [ig-reel, ig-caption]` when only those two plus `ig-human` are installed will fail review. Manually intersect with what actually exists in `~/.hermes/skills/`.
- **Never use `author: Hermes Agent` alone for a ported skill.** The original author is a real human; credit them first (e.g. `Jake Schincariol (Jakeschincariol), Hermes Agent`). This is a hardline repo rule that reviewers enforce.
- **`read_file` on Termux can return 0 chars for files that clearly have content.** Fall back to `with open(path) as f: f.read()` directly — the read_file wrapper has had silent-empty bugs on skill SKILL.md files specifically. If content looks empty but `wc -c` shows bytes, use raw open.
- **Don't touch existing curated skills when porting.** If the port would collide with a bundled or pinned skill (e.g. `hermes-agent`, `hermes-agent-skill-authoring`), install under a different name or skip. `skill_manage(action='patch')` on those will be refused.
- **`related_skills` self-reference is a common footgun.** When you write `related_skills: [<this-skill>, <other>]` in the port, it looks valid but is a review flag — every entry must resolve to a *different* existing skill.
- **Some repos use `sub-skills/` instead of `references/`.** Hermes doesn't have a `sub-skills/` concept — flatten those into `references/` with the filename prefixed (`references/post-audit.md` instead of `sub-skills/post-audit.md`).
- **The skill loader caches at session start.** A just-created skill cannot be `skill_view`-ed in the same turn. Verify by inspecting the file on disk instead.
- **Verify the repo actually exists before trusting promotional posters.** AI-generated "top open source tools" graphics hallucinate star counts, descriptions, and sometimes the whole repo. Before porting any URL from a poster, run `curl -s api.github.com/repos/<owner>/<name>` and check `full_name` is present. Compare the returned `description` to what the poster claims — mismatches mean the poster is AI-generated even if the repo exists. If `message: Not Found`, drop the port entirely.
- **Some repos ship `SKILL.md` files that are documentation for a pip-installable package, not self-contained skills.** Symptoms: frontmatter contains `dependencies.python`, `dependencies.pip`, or `mcp.command` fields; body says `pip install <pkg>` and references a CLI command like `<pkg>-mcp`. Examples: `vibe-trading` (SKILL.md documents the `vibe-trading-ai` PyPI package and its `vibe-trading-mcp` binary), `voicestudio` (skills describe an HTTP backend the user must run locally). Porting these `SKILL.md` files alone gives you references to commands that don't exist on the machine. Before porting, check `pip show <pkg>` or `which <cli>` — if missing, either install the package first or explicitly tell the user in the reply that the skill is a stub until they do. Do not silently port a stub and pretend it works.

## Not This Skill

- Modifying an existing Hermes skill — use `skill_manage(action='patch')`.
- Writing a new Hermes skill from scratch — see `hermes-agent-skill-authoring`.
- Installing bundled or hub skills from the Hermes skill marketplace — use `hermes skills install`.
- Ports that require a paid API key to function — port the files, but tell the user in the reply which specific lib/client needs the key so they can decide whether to keep it.
