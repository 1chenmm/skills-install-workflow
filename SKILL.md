---
name: skills-install-workflow
description: 技能安装7步工作流：搜索→去重→安全审查→安装→学习→复查→启用。含常用技能安装链接表。适用于从skills.sh安装任何技能。
version: 2.6.0
metadata:
  hermes:
    tags: [skills, workflow, install, security, dedup]
---

# 技能安装工作流

从 skills.sh 安装技能的标准化7步流程。每一步都有明确的命令和检查点。

---

## ① 搜索 — 双仓库搜索 (skills.sh + clawhub.ai)

**优先加载 `find-skills` 技能 + 同时搜 skills.sh 和 ClawHub。** ClawHub 的安装量更高、技能更成熟，优先从中选。

```bash
# 步骤1: 加载 find-skills 技能获取搜索方法
skill_view(name="find-skills")

# 步骤2: 用 npx skills find 搜 skills.sh
npx skills find "<关键词>"

# 步骤3: 用 clawhub search 搜 ClawHub（技能更丰富）
clawhub search "<关键词>"
```

**两个仓库对比：**

| 对比 | skills.sh (`npx skills`) | ClawHub (`clawhub search`) |
|------|-------------------------|---------------------------|
| 技能数量 | ~1万+ | 更丰富 |
| 安装量 | 一般几百~几千 | **几千到几万**（普遍更高） |
| 安全评估 | Gen/Socket/Snyk 三方 | 内置自动扫描 |
| 搜索方式 | `npx skills find <关键词>` | `clawhub search <关键词>` |
| 安装 | `npx skills add owner/repo@skill` | `clawhub install <slug>` |
| 前缀 | `owner/repo@skill` 格式 | `@author/slug` 格式 |

**为什么优先搜 ClawHub：**
- 安装量普遍更高（几万下载的技能很多）
- 同样是 Hermes 兼容的技能（SKILL.md + 脚本）
- `clawhub install` 更简洁（直接装到 `./skills/` 目录）

**检查点：**
- 从两个仓库各找 2-3 个候选技能
- 比较安装量、来源信誉、安全评估
- 优先选 ClawHub 中安装量高、信誉好的

**备选（无 find-skills 时回退）：**
```bash
curl -s "https://skills.sh/api/search?q=<关键词>&limit=10" | python3 -c "
import sys, json; data = json.load(sys.stdin)
for s in data.get('skills',[]):
    print(f\"名称: {s['skillId']} | ⬇{s.get('installs',0)} | 仓库: {s.get('source','')}\")
    print(f\"简介: {s.get('description','') or '(无)'}\")
    print()
"
```

---

## ② 去重 — 扫描本地技能库（不可跳过）

**第①步搜索后，用 `skills_list` 检查本地是否已有同名或类似功能技能**，如果已存在则跳过安装。

如果 find-skills 没有标记或对比不确定，手动执行：

### 2a. 列出所有本地技能

```bash
skills_list()
```

### 2b. 对每个可能相关的本地技能，读取完整 SKILL.md

```bash
skill_view(name="<本地技能名>")
```

### 2c. 逐项对比（不是模糊匹配，是实际字段对比）

| 对比维度 | 本地技能 | 候选技能 | 重合? |
|----------|---------|---------|-------|
| **触发词** | 哪些词触发 | 哪些词触发 | |
| **调用的工具/CLI** | 例如 `tavily`, `web_search` | 例如 `brave-search` | |
| **API/端点** | 例如 `api.tavily.com` | 例如 `api.search.brave.com` | |
| **核心功能** | 实际能做什么 | 实际能做什么 | |
| **输出格式** | 返回什么 | 返回什么 | |
| **外部依赖** | 需要什么Key/账号 | 需要什么Key/账号 | |

### 2d. 判定规则

```
如果 触发词重合 ≥ 60% 且 核心功能重合 ≥ 70%：
    → 🔁 重复，不装。告诉用户："你已有 <本地技能名>，功能和这个一样"
    
如果 核心功能重合 ≥ 50% 但 工具/API不同：
    → ⚠️ 部分重叠。列出差异让用户决定

如果 核心功能重合 < 30%：
    → ✅ 不重复，继续下一步
```

### 2e. 去重示例

```
搜索到: brave-search (Brave搜索API)
本地有: tavily-search (Tavily搜索API), anysearch (实时搜索引擎)

对比:
  tavily-search vs brave-search:
    触发词: "search"/"find"/"look up" vs "brave search" — 部分重叠
    工具: tavily CLI vs brave CLI — 不同
    核心功能: 网页搜索(LLM优化) vs 网页搜索 — 🟡 高度重叠
    → ⚠️ 功能重叠，但API不同。告诉用户已有tavily-search可满足搜索需求

  anysearch vs brave-search:
    触发词: 搜索相关 vs brave专用 — 低重叠
    核心功能: 多引擎搜索+URL提取 vs Brave单引擎 — 部分重叠
    → ⚠️ 有差异，anysearch更通用
```

---

## ③a 兼容性评估 — 检查来自其他生态的技能是否能在当前 Agent 运行

**不能只看描述就下结论！** 来自 OpenClaw、ClawHub、MCP.Directory 等生态的技能通常是纯 Python 脚本，跟宿主 Agent 没有硬绑定。

### 怎么做

1. **找到实际代码** — 从技能源拉取完整文件（不仅仅是 SKILL.md）：

```bash
# 用 clawhub 安装到临时目录查看
npx clawhub@latest install <技能名>  # 装到 /tmp/skills/<技能名>/

# 或从 GitHub raw 拉取
curl -sL "https://github.com/<owner>/<repo>/raw/main/skills/<name>/scripts/<file>"
```

2. **检查实际依赖** — 逐项核实：

| 检查项 | 通过条件 | 典型非OpenClaw依赖 |
|--------|---------|-------------------|
| Python 脚本 | `python3` + stdlib 或 `requests` | ✅ 大部分 Hermes 环境有 |
| API 调用 | HTTP POST/GET | ✅ 任何环境都能做 |
| `BAIDU_API_KEY` 等环境变量 | 可放 `~/.hermes/.env` | ✅ 跟其他 Key 一样管理 |
| OpenClaw 特有变量 | `DUMATE_SESSION_ID`, `DUMATE_SCHEDULER_URL` | ⚠️ 非 OpenClaw 环境不设，走正常路径 |

3. **运行试一下**（即使没有 API Key 也能验证代码结构是否正确）：

```bash
python3 /tmp/<skill>/scripts/search.py '{"query":"测试","count":3}'
# → 如果报"未设置 API Key"而非 ImportError/SyntaxError，说明代码本身能跑
```

### 兼容性判断表

| 现象 | 判定 | 操作 |
|------|------|------|
| 纯 Python 脚本，调用 HTTP API | ✅ **完全兼容** | 安装后用 `python3` 直接跑 |
| 需要特定 CLI（如 `clawhub`、`openclaw`） | ⚠️ 可能不兼容 | 看 CLI 是否能独立安装 |
| 调用 OpenClaw 内部 API/函数 | ❌ **不兼容** | 需重写或找替代方案 |
| 只是 SKILL.md（LLM 指令文档） | ✅ **完全兼容** | 装成 Hermes 技能即可 |
| Shell 脚本调用 curl | ✅ **完全兼容** | Hermes 有 terminal |

### 常见误解

- ❌ "这是 ClawHub 的技能，可能不兼容 Hermes" → ✅ 实际是 Python 脚本调用 HTTP API
- ❌ "需要 OpenClaw 才能跑这个脚本" → ✅ Python 脚本独立运行，不依赖 OpenClaw 运行时
- ❌ "SKILL.md 是给 Claude 看的" → ✅ 也是给 Hermes（我）看的指令文档

### 关键原则

> **先看代码，再下结论。** 不要因为来源生态不同就假设不兼容。
> 绝大多数第三方技能 = SKILL.md（LLM指令）+ 脚本（Python/Shell）= Hermes 原生兼容。

---

## ③b 安全审查 — 加载 skill-vetter + 自动扫描

**第一步：加载安全审查技能**

```bash
skill_view(name="skill-vetter")
```

> 这会加载 `skill-vetter` 安全审查技能的完整指令，包含红/黄旗检测、权限范围分析、可疑模式扫描。

**第二步：拉取候选技能的 SKILL.md**

```bash
curl -s "https://raw.githubusercontent.com/<owner>/<repo>/main/skills/<name>/SKILL.md" > /tmp/skill.md

# 或用 skills.sh 页面
web_extract(urls=["https://skills.sh/<owner>/<repo>/<name>"])
```

**第三步：用 skill-vetter 执行自动扫描**

**第四步：用 SkillSpector 执行深度安全扫描**

如果安装了 SkillSpector（NVIDIA AI 技能安全扫描器），在 skill-vetter 审查后额外跑一次：

```bash
# 扫描候选技能的本地目录
skillspector scan /tmp/staging/<技能名>/ --no-llm

# 或扫描已安装的技能目录
skillspector scan ~/.hermes/skills/<技能名>/ --no-llm
```

> `skillspector` 别名只在交互会话中有效。cron 或无别名环境用完整路径：
> `/root/miniconda3/envs/skillspector/bin/skillspector scan <目录> --no-llm`
> 
> `--no-llm` 跳过 LLM 语义分析（不需要 API Key），只跑静态规则扫描。  
> 如需 LLM 分析，设 `OPENAI_API_KEY` 或 `DASHSCOPE_API_KEY` 环境变量即可。

**SkillSpector 检查的额外风险维度：**

| 类别 | 说明 | 典型高危 |
|------|------|---------|
| AST 静态分析 | `exec()`/`eval()`/`subprocess`/`os.system` 调用 | 🔴 `exec()` / `eval()` → CRITICAL |
| 数据泄露 (E2) | 读取环境变量中的 API Key/Token | 🔴 发送到外网 → HIGH |
| 污点追踪 (TT) | 外部输入 → 代码执行的数据流 | 🔴 远程命令注入 → CRITICAL |
| YARA 签名 | 恶意软件/Webshell/挖矿脚本匹配 | 🔴 任意匹配 → CRITICAL |
| 提权 (PE3) | 读取 ~/.ssh / ~/.aws 等凭证路径 | 🔴 凭证访问 → HIGH |
| 供应链 (SC) | 远程拉取脚本、混淆代码、已知漏洞依赖 | 🔴 远程代码执行 → HIGH |

**风险评分参考：**

| 评分 | 等级 | 建议 |
|------|------|------|
| 0–20 | 🟢 LOW | 安全，可安装 |
| 21–50 | 🟡 MEDIUM | 谨慎，确认后安装 |
| 51–80 | 🔴 HIGH | 不建议安装 |
| 81–100 | ⛔ CRITICAL | 拒绝安装 |

**注意事项：**
- SkillSpector 扫描结果可能包含**误报**（如搜索脚本用 `subprocess` 是正常设计），需结合人工判断
- 重点关注 CRITICAL 级别的发现（`exec()`、恶意软件匹配、远程命令注入）

加载 `skill-vetter` 后，它会自动检查：

| 等级 | 模式 | 含义 |
|------|------|------|
| ⛔ | `curl\|sh` / `wget\|bash` | 远程脚本管道执行 — 拒绝 |
| ⛔ | `eval.*curl` / `base64\|sh` | 远程代码执行 — 拒绝 |
| ⛔ | `~/.ssh` / `~/.aws` / `~/.gpg` | 读取密钥文件 — 拒绝 |
| ⛔ | 硬编码 API key / token | 明文泄露 — 拒绝 |
| ⚠️ | `.env` / `API_KEY` / `TOKEN` | 索要凭证 — 警告 |
| ⚠️ | `paste.*api.key` / `enter.*token` | 索要凭证 — 警告 |
| ⚠️ | `allowed-tools: Bash(*)` | 无限制命令 — 警告 |
| ⚠️ | `docker` / `sudo` / `chmod 777` | 提权操作 — 警告 |
| 🟢 | 纯文档技能 / 无外部依赖 | 安全 — 通过 |

**第五步：手动补充 grep 扫描（快速确认）**

```bash
grep -cE '(curl|wget).*\|.*(ba)?sh' /tmp/skill.md    # 必须=0
grep -cE 'eval.*(curl|wget)' /tmp/skill.md            # 必须=0
grep -cE 'base64.*\|.*sh' /tmp/skill.md               # 必须=0
grep -cE 'sudo|chmod 777' /tmp/skill.md               # 必须=0
```

**风险判定：**
- 🟢 LOW → 无红黄旗 → 安装OK
- 🟡 MEDIUM → 有黄旗 → 读代码后决定
- 🔴 HIGH → 有红旗 → 不装，告诉用户

---

## ④ 安装

根据技能来源选择安装方式：

### 方式 A：skills.sh 安装（默认）

```bash
npx -y skills add <owner>/<repo> --skill <name> -g --yes
```

参数说明：
- `-g` — 全局安装（用户级）
- `--yes` — 跳过交互确认
- `--skill <name>` — 指定安装哪个技能（仓库可能含多个）

安装后文件位置：
- `~/.agents/skills/<name>/SKILL.md`
- `~/.hermes/skills/<name>` → symlink 到上面

### 方式 B：ClawHub 安装（推荐，技能更丰富）

先通过 clawhub search 找到技能 slug，然后安装：

```bash
# 搜索
clawhub search "<关键词>"

# 安装（slug 格式：@作者/技能名 或 直接技能名）
clawhub install <slug>

# 示例
clawhub install @barneyjm/search-2
```

**ClawHub 安装特点：**
- 装到当前目录的 `./skills/<name>/` 下
- 自动生成 `.clawhub/lock.json` 锁定文件
- 无 `-g` 全局参数，需手动搬到 Hermes 技能目录

**安装后搬到 Hermes：**

```bash
# 确保在目标仓库目录下操作（如 /tmp/<repo>）
cd /tmp/<repo>
clawhub install <slug>

# 检查安装结果
ls -la skills/<name>/

# 搬到 Hermes 技能目录
mkdir -p ~/.hermes/skills/<name>/
cp -r skills/<name>/* ~/.hermes/skills/<name>/
rm -rf skills/<name>/

# 执行 /reload-skills 刷新
```

> **为什么 ClawHub 优先：** ClawHub 的技能安装量普遍比 skills.sh 高一个数量级，社区更活跃，技能质量经过更多验证。同一功能的技能，ClawHub 的往往是更成熟的选择。

### 方式 C：GitHub 直接拉取

当技能不在 skills.sh 也不在 ClawHub 时（如字节跳动官方 byted-web-search）：

```bash
git clone --depth 1 <仓库URL> /tmp/staging
cp -r /tmp/staging/skills/<名称>/* ~/.hermes/skills/<名称>/
rm -rf /tmp/staging
```

**注意：** 无论哪种安装方式，装完后都要创建 Hermes 兼容的 SKILL.md（带有 `---` frontmatter 和 `credentials` 字段）。

### 方式 E：安装 CLI 工具

当用户说的是安装一个**命令行工具**（如 `clawhub`、`openclaw`、`npx` 等），而不是 Hermes 技能时：

```bash
# npm 全局安装
npm i -g <包名>

# 验证安装
<cli-name> --help

# 检查登录状态（如适用）
<cli-name> whoami
```

**如何区分用户要的是 skill 还是 CLI 工具：**
- "安装 clawhub" → 先查文档（官方链接/docs）→ 如果是 `npm i -g` 的 CLI，走方式 E
- "安装 xx 技能" → 搜 skills.sh / clawhub.ai → 走方式 A/B
- 给的是官方文档链接 → 按文档指示安装

**装完 CLI 工具后：**
- 记下版本号、登录状态、搜索命令
- 更新 `skills-install-workflow` 的搜索步骤（如果是搜索相关的 CLI）

**推送前先加载 `github-push` 技能，它内置完整的敏感信息扫描和 git 历史审计流程：**

```bash
skill_view(name="github-push")
```

该技能会按 6 步流程执行：准备文件 → 敏感扫描（API Key/邮箱/手机/IP/SSH/账号）→ git 历史审计 → 文件审核 → 安全推送 → 推送后复查。

如果 `gh` 已登录但 `git push` 报 auth 错误（头服服务器常见）：

```bash
git remote set-url origin \
  "https://x-access-token:$(gh auth token)@github.com/<user>/<repo>.git"
git push
```

> 详见 `references/github-push-auth.md`

---

## ⑤ 学习 — 加载技能内容

```bash
skill_view(name="<技能名>")
```

**重点看：**
- `allowed-tools:` — 权限范围
- 依赖的CLI/工具（`npm i -g xxx` / `pip install xxx`）
- API Key / Token / 登录要求
- 触发词（什么时候自动激活）
- 用法示例和路由逻辑

---

## ⑥ 复查 — 依赖是否到位

```bash
# 检查CLI是否安装
which <cli-name> 2>/dev/null && echo "✅ 已安装" || echo "❌ 未安装"

# 检查Token/认证文件
[ -f ~/.config/<name>/token.json ] && echo "✅ 已登录" || echo "❌ 需登录"

# 检查API Key环境变量
[ -n "$API_KEY" ] && echo "✅ 已设置" || echo "❌ 需设置"

# 检查是否需要付费/余额
# (查看SKILL.md中的价格说明)
```

**输出依赖报告：**
```
📦 CLI:       ✅ / ❌
🔑 Token:     ✅ / ❌  
💰 付费:      是/否 ($x.xx/次)
🌐 网络:      需要/不需要
```

---

## ⑦ 启用 — 加载到会话

安装后需要重新扫描技能目录：

```
在对话中输入: /reload-skills
或重启 Hermes: hermes --skills <name>
```

技能会在匹配触发词时自动激活。如需手动加载：
```
skill_view(name="<技能名>")
```

---

## 常用技能安装链接

以下是从 skills.sh 安装本工作流相关技能的推荐命令：

| 技能 | 用途 | 安装命令 |
|:----:|:----|:---------|
| 🔍 **find-skills** | 跨仓库搜索技能 | `npx -y skills add vercel-labs/skills --skill find-skills -g --yes` |
| 🛡️ **skill-vetter** | 安全审查 | `npx -y skills add useai-pro/openclaw-skills-security --skill skill-vetter -g --yes` |
| 🔄 **agent-workflow** | 通用任务处理工作流 | `npx -y skills add ruvnet/ruflo --skill agent-workflow -g --yes` |
| 🐛 **systematic-debugging** | 四阶段根因调试 | `npx -y skills add obra/superpowers --skill systematic-debugging -g --yes` |

> 🆕 找不到某个技能时，先用 `find-skills` 搜索最新版本。

## 完整速查

```bash
# 1. 搜索（双渠道）
# 渠道A: skills.sh
skill_view(name="find-skills")
npx skills find "<关键词>"
# 渠道B: clawhub.ai
clawhub search "<关键词>"
# → 搜两个地方，互相补充

# 2. 去重（找到已安装标记则跳过）
# → 对未标记的技能逐个 skill_view 对比触发词、工具、API、核心功能

# 3a. 兼容性评估 — 拉取实际代码，检查依赖和运行时兼容性
# → 不因来源生态不同就假设不兼容，先看实际脚本

# 3b. 安全审查（加载 vetter + SkillSpector 深度扫描）
skill_view(name="skill-vetter")
# → 拉取 SKILL.md 后自动审查红/黄旗
skillspector scan ~/.hermes/skills/<name>/ --no-llm
# → 额外检测 AST/污点追踪/YARA签名等64种模式

# 4. 安装（ClawHub 推荐）
clawhub install <slug>                        # ClawHub方式（优先）
# 或 npx -y skills add <owner>/<repo> --skill <name> -g --yes  # skills.sh方式

# 5. 学习
skill_view(name="<name>")

# 6. 复查依赖
which <cli>; [ -f ~/.config/<name>/token.json ]; echo $API_KEY

# 7. 启用
/reload-skills
```

---

## 注意事项

- **skills.sh 安装的技能**用 `hermes skills uninstall` 删不了，直接 `rm -rf ~/.agents/skills/<name> ~/.hermes/skills/<name>`
- **第①步优先用 find-skills** 搜索，不要只查 skills.sh。加载后按技能指示操作。
- **第②步去重不可跳过** — 名字不像≠功能不重叠。find-skills 已自动标记已安装技能，但对未标记的必须读 SKILL.md 代码对比
- **第③b步安全不可跳过** — 技能运行在 agent 权限下，能执行命令、读写文件。先用 `skill-vetter` 自动扫描，再用 `skillspector` 深度扫描 + 手动 grep 确认
- **付费技能** 确认余额后再用，避免意外扣费
- **外部依赖**（CLI/Token）不满足就别装，装了也用不了

## ⚠️ 关键陷阱（Pitfalls）

### 陷阱 0：听到"技能"两个字就先加载本技能

用户说"搜索技能""有没有技能""装个技能""找技能"——**第一个动作不是搜索，是 `skill_view(name="skills-install-workflow")`**。

不加载=跳过流程=用 web_search 瞎搜=用户纠正你=浪费时间。本技能的 pitfalls 只在加载后才生效，所以必须先加载。

### 陷阱 1：用户问"有没有技能"时，必须用这个工作流

当用户问以下任何一种问题时，**不要手动搜索，必须用这个工作流**：
- "找一下有没有 XX 技能"
- "有没有做 XX 的技能"
- "我想装一个 XX 技能"
- "搜索技能"
- "你会打电话吗"（暗示需要技能扩展）
- 任何关于技能安装/查找/评估的问题

**错误做法（用户会纠正你）：**
```
用户："你会打电话吗"
错误：直接用 web_search 搜"phone skill"
用户："你用上你安装技能的那个工作流了吗？"
→ 用户发现你没用工作流，觉得"笨笨的"
```

**正确做法：**
```
用户："你会打电话吗"
正确：
  1. skills_list() 扫描本地技能库
  2. 发现没有电话技能
  3. 用 find-skills 跨仓库搜索 "phone call"
  4. 按工作流去重 → 安全扫描 → 给出候选列表
  5. 问用户要装哪个
```

### 陷阱 2：扫描技能库 ≠ 搜索网页

`skills_list()` 扫描的是**本地已安装的技能**，不是互联网搜索。
如果本地没有，下一步是 `find-skills` 跨仓库搜索，不是 `web_search`。

### 陷阱 3："你定"不等于"随便"，用户期望你遵循流程

当用户说"你定"、"你决定"时，**仍然必须扫描技能库**。
用户说"你定"的前提是信任你会按流程做事。
跳过流程 = 辜负信任 = 用户说"笨笨的"。

### 陷阱 4：技能安装后不复查

装了技能不代表能用。必须检查：
- CLI 是否安装（`which xxx`）
- API Key 是否配置（`env | grep KEY`）
- 是否付费（查看余额）

### 陷阱 6：API Key 含特殊字符 → shell echo 写入会炸

**来源：** 会话 2026-06-09，用户提供的火山引擎和百度 API Key 含特殊字符

当 API Key 含有 `!`、`$`、`\`、`#`、`&`、`'`、`"` 等特殊字符时：

❌ **错误做法：**
```bash
echo 'API_KEY=abc$def!ghi' >> .env    # $def 被当成变量展开
```

✅ **正确做法（任选其一）：**
```bash
# 方式 1：用 write_file 工具（推荐，自动处理特殊字符）
# 用 skill_manage(write_file) 或直接 write_file()

# 方式 2：用 Python 写文件
python3 -c "
key = '这里直接粘贴Key字... with open('/path/to/.env', 'a') as f:
    f.write('VAR_NAME=*** + key + '\n')
"

# 方式 3：先写 Python 脚本文件，再执行
cat > /tmp/write_key.py << 'PYEOF'
key = '原始API... with open('/path/to/.env', 'a') as f:
    f.write('VAR_NAME=*** + key + '\n')
PYEOF
python3 /tmp/write_key.py
```

### 陷阱 8：替换已安装技能时忘了删旧版

**来源：** 会话 2026-06-28，替换 find-skills (agentspace-find-skills → vercel-labs/skills)

当需要替换一个已安装的技能（如从旧版换到新版/换来源）时：

1. **先查旧版位置：** `find ~/.agents/skills/<旧名>/ ~/.hermes/skills/<旧名>/ -type f`
2. **手动删除旧的：**
   ```bash
   rm -rf ~/.agents/skills/<旧名> ~/.hermes/skills/<旧名>
   ```
3. **再装新的**（否则 npx skills add 可能冲突）
4. **执行 `/reload-skills`** 刷新列表
5. **验证旧版消失：** `skills_list()` 确认不再出现

### 陷阱 9：skills.sh 搜出来的技能"太拉了" — 预期管理

**来源：** 会话 2026-06-28，用户评价搜索结果的反馈

skills.sh 的搜索结果经常出现安装量低（<100）、作者未知、描述简略的技能。这**不是你的问题**，是公开技能库的普遍现状。不要因此反复换关键词搜，或推荐明显低质量的技能。

**正确做法：**
1. 承认现状："公开技能库质量参差不齐，这个方向没有特别好的现成技能"
2. 换 **ClawHub 再搜一轮**：`clawhub search "<关键词>"`
3. 如果两个渠道都没有满意的 → **"要不我直接给你写一个？"**

**错误做法：**
- 捏造安装量或好评来让技能显得更好
- 推荐你肉眼都看不上的技能
- 反复换关键词搜十几次还不放弃

### 陷阱 10：用户说"安装 clawhub"不一定是安装 Hermes 技能

**来源：** 会话 2026-06-28

"clawhub" 既可以是一个 skills.sh 上的 OpenClaw 技能，也是一个 `npm i -g clawhub` 的 CLI 工具。用户给官方文档链接（docs.openclaw.ai/clawhub）时就说明是 CLI 工具。

**判断方法：**
- 用户只说名字 → 先搜 skills.sh，如果搜到的是 OpenClaw 专用的（命令是 `openclaw skills xxx`），问用户要做什么
- 用户给官方文档链接 → 按文档装 CLI 工具
- 不确定时直接问用户："你是要装一个 CLI 工具还是一个技能？"

**不要用 `hermes skills uninstall` 卸载 skills.sh 安装的技能** — 那个命令只对 Hermes 原生技能有效。skills.sh 的技能用 `rm -rf` 物理删除。

**来源：** 会话 2026-06-09，安装 baidu-search 和 byted-web-search

从 ClawHub 或 GitHub 直接拉取的技能默认装到临时目录（`/tmp/skills/` 或 git clone 目录），不会自动出现在 Hermes 技能列表里。装完后必须：

1. **搬脚本文件**到 `~/.hermes/skills/<名称>/scripts/`
2. **创建 Hermes 兼容 SKILL.md**（带 `---` frontmatter、`credentials` 字段）
3. **写 API Key** 到 `~/.hermes/.env` 或技能目录下的 `.env`
4. **测试验证**（直接跑脚本看是否能返回正确结果）
5. **清理临时文件**（`rm -rf /tmp/skills/ /tmp/staging`）

## 参考文件

- `references/skillspector-usage.md` — SkillSpector 详细安装/使用/误报判断指南
- `references/github-readme-format.md` — GitHub 开源时的 README 格式规范（中英文分开、Star 趋势图底部、脱敏）
- `references/github-push-auth.md` — 用 `gh auth token` 解决 headless server 上 git push 的认证问题
- `references/repo-sensitive-audit.md` — 开源前审计仓库：扫描 git 历史邮箱、源码中的 API Key/Token/手机号/QQ，修复 filter-branch 残留
- `references/skills-install-workflow-cleanup.md` — 技能库清理完整案例（从 105 到 63 的实操记录）

## 相关技能

- `skill-library-maintenance` — 技能库审计、清理、去重、配置漂移修复
- `skill-cleaner` — 审计 prompt budget 和重复
- `skill-vetter` — 安全审查
- `find-skills` — 跨仓库搜索
- `github-push` — 推送前敏感信息扫描 + git 历史审计

- **配置修复模式** — 如果 model 配置被误修改（如正在切换 provider 时的回滚），检查 `hermes config` 确认并用 `python3` 直接修改 yaml 避免 `hermes config set` 的类型转换问题
- **不要让 Agent 自动跑 ark-helper** — 用户偏好手动操作 TUI，ark-helper 的 TUI 交互太复杂，让用户自己去终端跑
- **退订并删除旧 provider 配置** — 用户退订后，必须完整删除旧 provider 的 API key、base_url、模型列表，从 `.env` 和 `config.yaml` 两边都删干净
