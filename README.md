# skills

[![skills.sh](https://skills.sh/b/magician336/skills)](https://skills.sh/magician336/skills)

个人 Agent skills 集合。每个 skill 是一个独立目录，入口文件固定为：

```text
skills/<skill-name>/SKILL.md
```

这些 skill 面向**真实工程**：把「怎么评审一份计划」「怎么判断 agent 真的做完了」「怎么评估一块代码的结构健康度」这类容易做糊的判断，写成可复用的纪律，而不是靠临场发挥。

## 安装

### 通用（推荐，支持 80+ 个 agent）

```bash
npx skills add magician336/skills
```

安装器会问你装哪些 skill、装到哪个 agent。也可以精确指定：

```bash
npx skills add magician336/skills --skill plan-review -a claude-code
```

| 命令 | 作用 |
| --- | --- |
| `npx skills find` | 搜索可安装的 skill |
| `npx skills list` | 列出已安装的 skill |
| `npx skills check` | 检查更新 |
| `npx skills update` | 更新已安装的 skill |
| `npx skills use magician336/skills@plan-review` | 不安装，直接把 skill 变成 prompt |

> 需要 Node.js 22.20 或更高版本。安装到全局（所有项目可用）加 `-g`。

### Claude Code

```bash
claude plugin marketplace add magician336/skills
claude plugin install magician336-skills@magician336
```

插件会**自动更新**。

### Codex

```bash
codex plugin marketplace add magician336/skills
codex plugin add magician336-skills@magician336
```

启动时自动更新。

### GitHub Copilot（CLI 与 VS Code）

```bash
copilot plugin marketplace add magician336/skills
copilot plugin install magician336-skills@magician336
```

然后在 `~/.copilot/settings.json` 里加一次：

```json
{
  "extraKnownMarketplaces": {
    "magician336": { "source": { "source": "github", "repo": "magician336/skills" }, "autoUpdate": true }
  }
}
```

VS Code 里运行 `Chat: Install Plugin From Source`，填 `https://github.com/magician336/skills`。

### Gemini CLI（手动更新）

```bash
gemini skills install https://github.com/magician336/skills.git --path skills
```

更新时重跑同一条命令。

## Skills 索引

### 评审纪律

| Skill | 干什么 | 什么时候用 |
| --- | --- | --- |
| [plan-review](./skills/plan-review/SKILL.md) | 评审还没开工的计划/规格/设计：锁定目的地、验收矩阵完整性、不可变契约、并行面所有权 | 「这个设计能不能开工」「计划有没有漏洞」 |
| [implementation-review](./skills/implementation-review/SKILL.md) | 评审 agent 刚写完的代码：逐行追实现接缝与证据、验证据有效性、出交付结论 | 「代码写完了帮我看看」「能不能合了」 |
| [architecture-health-review](./skills/architecture-health-review/SKILL.md) | 结构健康度审查：先找缺陷 → 再按固定锚点打分 → 给模块级重构方向（带置信度） | 「这套结构有什么问题」「哪里该重构」 |

### 设计哲学与工程实践

| Skill | 干什么 | 什么时候用 |
| --- | --- | --- |
| [ousterhout-software-design](./skills/ousterhout-software-design/SKILL.md) | 用 Ousterhout《软件设计的哲学》的复杂性、模块深度、信息隐藏与演化原则做设计、重构与评审 | 讨论模块边界、深浅模块、抽象层次时 |
| [jinguantao-kongzhilun](./skills/jinguantao-kongzhilun/SKILL.md) | 用《控制论与科学方法论》的控制、反馈、系统演化、质变与黑箱模型分析问题 | 分析系统性问题、反馈回路时 |
| [writing-windows-scripts](./skills/writing-windows-scripts/SKILL.md) | 写/改 Windows `.bat`/`.cmd`/`.ps1` 时的检查清单，专治中文乱码与 cmd 解析器炸裂 | 交付含 Windows 脚本的改动前 |

## 新增一个 skill

1. 建目录 `skills/<skill-name>/`；
2. 写 `SKILL.md`，frontmatter 必须有 `name`（小写连字符，与目录同名）和 `description`；
3. `description` 决定它**什么时候被触发**——要写清楚能力、触发短语、以及**哪些相邻请求不该触发它**（避免和仓库里已有的 skill 抢触发）；
4. 正文只写执行规则（流程、输出契约、失败处理），大块素材拆到 `references/<topic>.md`，主文件用相对链接引用；
5. 在 `.claude-plugin/plugin.json` 的 `skills` 数组里登记它。

`name` 与目录名不一致会被校验工具判失败。

## License

[MIT](./LICENSE)
