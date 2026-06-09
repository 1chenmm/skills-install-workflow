# 🔧 Skills Install Workflow

> A 7-step security-first workflow for installing AI agent skills from any registry (skills.sh, clawhub.ai, GitHub).

[![Stars](https://img.shields.io/github/stars/1chenmm/skills-install-workflow?style=social)](https://github.com/1chenmm/skills-install-workflow)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## ⭐ Star History

[![Star History Chart](https://api.star-history.com/svg?repos=1chenmm/skills-install-workflow&type=Date)](https://star-history.com/#1chenmm/skills-install-workflow&Date)

---

## 🇨🇳 中文版

**技能安装工作流** — 一个标准化的 7 步流程，用于安全地安装 AI Agent 技能。

### 核心思想

- **跨仓库搜索**：不只查 skills.sh，同时查 clawhub.ai + GitHub
- **代码级去重**：不是看名字，而是读 SKILL.md 对比触发词、CLI、API、核心功能
- **安全审查**：加载 `skill-vetter` 自动扫描 + 手动 grep 确认
- **依赖复查**：装之前确认 CLI / Token / 付费条件都到位

### 7 步流程

| 步骤 | 动作 | 关键检查 |
|------|------|---------|
| ① 搜索 | 跨仓库搜索 + 安全扫描 | 找到 2-3 个候选，记下风险等级 |
| ② 去重 | 扫描本地技能库 | 代码级对比，判定重复/部分重叠/不重复 |
| ③ 安全审查 | 加载 skill-vetter + 自动扫描 | 红/黄旗检测，grep 确认 |
| ④ 安装 | `npx skills add` | 指定 `-g --yes` |
| ⑤ 学习 | `skill_view(name)` | 看权限、依赖、触发词、用法 |
| ⑥ 复查 | 检查 CLI / Token / 付费 | 不满足就别装 |
| ⑦ 启用 | `/reload-skills` | 重启后自动激活 |

### 完整速查

```bash
# 1. 搜索（跨仓库）
bash ~/.hermes/skills/agentspace-find-skills/scripts/find.sh "<关键词>" --scan 2

# 2. 去重
# → 对比触发词、工具、API、核心功能

# 3. 安全
skill_view(name="skill-vetter")

# 4. 安装
npx -y skills add <owner>/<repo> --skill <name> -g --yes

# 5. 学习
skill_view(name="<name>")

# 6. 复查依赖
which <cli>; [ -f ~/.config/<name>/token.json ]; echo $API_KEY

# 7. 启用
/reload-skills
```

---

## 🇺🇸 English Version

**Skills Install Workflow** — A standardized 7-step security-first process for installing AI agent skills.

### Core Principles

- **Cross-registry search**: Search skills.sh, clawhub.ai, and GitHub simultaneously
- **Code-level deduplication**: Compare trigger words, CLI tools, API endpoints, and core functionality — not just names
- **Security review**: Load `skill-vetter` for automated scanning + manual grep confirmation
- **Dependency check**: Verify CLI / Token / payment requirements before installing

### 7-Step Workflow

| Step | Action | Key Check |
|------|--------|-----------|
| ① Search | Cross-registry search + security scan | Find 2-3 candidates, note risk level |
| ② Deduplicate | Scan local skill library | Code-level comparison to detect overlap |
| ③ Security | Load skill-vetter + auto-scan | Red/yellow flag detection, grep confirmation |
| ④ Install | `npx skills add` | Use `-g --yes` |
| ⑤ Learn | `skill_view(name)` | Check permissions, dependencies, triggers |
| ⑥ Verify | Check CLI / Token / billing | Skip if requirements not met |
| ⑦ Enable | `/reload-skills` | Auto-activate after restart |

### Quick Reference

```bash
# 1. Search (cross-registry)
bash ~/.hermes/skills/agentspace-find-skills/scripts/find.sh "<keyword>" --scan 2

# 2. Deduplicate
# → Compare trigger words, tools, API, core functionality

# 3. Security
skill_view(name="skill-vetter")

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

## 📁 文件结构

```
.
├── README.md          # 本文件（中英文 + Star 趋势图）
├── SKILL.md           # 完整工作流文档（可直接用于 Hermes Agent）
└── LICENSE            # MIT 许可证
```

---

## ⚡ 适用场景

- 安装 AI Agent 技能（如 Hermes Agent、Claude Code、OpenClaw 等）
- 需要在多个技能仓库间搜索并对比
- 要求安全审查，防止安装恶意技能
- 管理技能库，避免重复安装功能重叠的技能

---

## 🤝 致谢

- 基于 [skills.sh](https://skills.sh) 技能注册表
- 集成 [agentspace-find-skills](https://skills.sh/agentspace-so/skills/find-skills) 跨仓库搜索
- 安全审查由 [skill-vetter](https://skills.sh) 提供

---

## 📜 License

MIT License — 详见 [LICENSE](LICENSE) 文件。
