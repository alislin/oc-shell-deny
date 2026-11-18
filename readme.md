# opencode-shell-deny

为 OpenCode 配置**可分档的全局限制**：修改源代码文件时优先使用内置 `edit` / `write` 工具，
限制通过 shell 命令（PowerShell cmdlet、重定向、`sed -i` 等）直接改写文件。
实现方式 = 硬限制（permissions deny/ask）+ 软性指令（全局 AGENTS.md），
提供**三档约束强度 + 卸载**，四态可随时切换；代理执行时由用户选择档位。

## 方案概览（四态模型）

| 状态 | AGENTS.md 标记 | permissions 构成 | 规则数 | 适用 |
|---|---|---|---|---|
| 卸载 | 无标记块 | 无本方案规则 | 0 | 不再需要任何约束 |
| 档 1 软性约束 `soft` | `mode=soft` | 仅 allow 基线 | 1 | 信任模型遵守、追求零打扰 |
| 档 2 优化折衷 `balanced`（**推荐**） | `mode=balanced` | allow + 精准 deny 矩阵 + 重定向 ask | 136 | 日常效率与源码防护的平衡点 |
| 档 3 最高限制 `strict` | `mode=strict` | 全量拦截写文件命令 | 18 | 最强编码防护、零判断成本 |

防护光谱：**卸载 ← 档 1 ← 档 2 ← 档 3**（效率递减、防护递增）。

各档定位：

- **档 3（strict）**：拦截一切 shell 写文件方式（含大小写变体），任何带 `>` 重定向都需人工确认。
  硬限制最彻底，但写日志、生成产物也会被打断。
- **档 2（balanced）**：只拦「写文件方式 × 源码扩展名」的组合；写日志（`npm run build > build.log`）、
  生成非源码产物完全放行；重定向写源码才弹确认。
  代价：黑名单基于命令文本匹配，变量间接写法拦不住（靠软指令约束）。
- **档 1（soft）**：无任何拦截，完全依赖 AGENTS.md 软指令。
  零误拦零弹窗，但违反时没有任何反馈信号（如编码损坏会静默发生）。
- **卸载**：移除本方案全部痕迹，恢复部署前状态。

## 适用范围

- 操作系统：Windows（含中文环境，PowerShell 编码问题见「编码要求」）
- Shell：PowerShell（Windows PowerShell 5.1 与 PowerShell 7 均适用）
- 配置对象：OpenCode 全局配置，对所有项目生效

## 如何使用（新主机部署）

把本仓库的 GitHub 链接发给代理（AI），并说明"按此文档完成配置"即可。
本文档是写给代理阅读并执行的说明书，**不提供也不需要任何安装脚本**。
代理会先检测当前状态并向你展示档位菜单，经你确认后才动手。

## 代理职责

1. 通读本文档全文，理解每一步之后再动手；
2. 环境与状态检测：
   - `opencode.json` 是否已存在、是否已有 `permissions`（并识别其中用户自有的规则）
   - `AGENTS.md` 是否存在、是否含本方案标记块（`oc-shell-deny:start`，读出当前档位）
   - 是否为旧版单档方案部署（特征：AGENTS.md 含"文件修改规则（强制）"但无标记块，
     且 permissions 含旧版 18 条规则）
3. 向用户整理汇报"要做的事情"，内容包括：环境与状态检测结果、四态菜单
   （**默认推荐档 2**，附各档一句话权衡）、将新建/修改哪些文件、备份文件名；
   然后明确询问用户选择目标档位，**未经用户确认，不得开始任何改动**；
4. 遇到不确定的情况（已有配置冲突、异常报错、路径不存在等）先询问用户，不要擅自决定；
5. 按用户确认后的「实施步骤」顺序执行所选档位，不得跳步；
   执行中途用户要求改档位时，按「档位切换与卸载」流程处理；
6. 完成后按「汇报」一节的要求输出结果（必须包含当前档位）。

## 三档拦截内容

### 档 3 最高限制（strict）

| 命令 / 模式 | 效果 | 说明 |
|---|---|---|
| `Set-Content` / `Add-Content` / `Out-File` / `Clear-Content` / `Tee-Object`（含小写变体） | deny | PowerShell 写文件 cmdlet |
| `[IO.File]::WriteAllText()` / `WriteAllLines()` / `AppendAllText()` / `WriteAllBytes()` | deny | 绕过 cmdlet 的 .NET 写文件 API |
| `sed -i` / `tee` / `Tee-Object`（含小写变体） | deny | 类 Unix 原地编辑、管道写入 |
| 任何含 `>` 输出重定向的命令 | ask | 无法区分"写日志"与"改源码"，交人工确认 |
| 其余所有操作 | allow | 构建、测试、git、包管理、只读查询不受影响 |

### 档 2 优化折衷（balanced）

拦截逻辑 = **写文件方式 × 源码扩展名**：

| 维度 | 内容 |
|---|---|
| 源码扩展组（15） | `*.py*`（连带 .pyc）、`*.js*`（连带 .json）、`*.ts*`（连带 .tsx）、`*.c*`（连带 .cpp/.cs/.css）、`*.h*`（连带 .hpp/.html）、`*.go*`、`*.rs*`、`*.java*`、`*.sh*`、`*.ps1*`、`*.sql*`、`*.rb*`、`*.php*`、`*.swift*`、`*.kt*` |
| 写文件方式（8） | `Set-Content`、`Add-Content`、`Out-File`、`Clear-Content`、`Tee-Object`、`sed -i`、`*WriteAll*`（一条覆盖 WriteAllText/Lines/Bytes）、`*AppendAllText*` |

| 操作 | 效果 | 说明 |
|---|---|---|
| 写方式 × 源码扩展（如 `Set-Content app.py`） | deny | 硬拒绝，AI 被拒后应立即改用内置工具 |
| 重定向落点为源码（如 `echo x > app.py`，含 `>>`、`2>`） | ask | 交人工确认 |
| `npm run build > build.log`、写日志/临时文件/非源码产物 | allow | 完全放行，零打扰 |
| `python -c` / `node -e` 等解释器一行式写文件 | allow | 不设规则，由 AGENTS.md 软指令约束 |
| 只读操作（如 `Get-Content app.py`） | allow | 不受影响 |

与档 3 的三点差异（有意设计）：

1. **不列大小写变体**——AI 生成的 PowerShell 命令几乎总是官方大小写；
   小写变体靠 AGENTS.md 软指令兜底，用户可按需自行追加变体规则；
2. **`.md` / `.yml` / `.toml` 等文档与配置扩展名不拦**——属性争议大、误伤多，由软指令覆盖；
3. **变量间接写法（`$f = "a.py"; Set-Content $f`）匹配不到**——软指令明令禁止此规避手法。

### 档 1 软性约束（soft）

无任何硬拦截。全部约束来自 AGENTS.md 文案（见实施步骤 3 的档 1 版本）。

## 背景知识（官方机制，勿凭猜测）

- OpenCode 全局配置：`~/.config/opencode/opencode.json`
  （Windows 即 `C:\Users\<用户名>\.config\opencode\opencode.json`），对所有项目生效。
  全局 `permissions` 规则在项目规则之后追加、"最后匹配优先"，
  因此全局 deny 优先级高于任何项目级配置，且不可被 "Allow always" 审批覆盖。
- `shell` 权限规则的 `resource` 是 **scanner 产生的命令字符串**
  （复合命令会拆成多个子命令逐一检查），匹配为整值通配：`*` 零或多字符（含 `/`）、
  `?` 恰一字符、其余字面量；模式以 `" *"` 结尾时也匹配无参数形态。
  **它匹配的是命令文本，不是目标文件路径**——因此"只按目标文件是否为源码"来拦截
  只能做到近似（依据命令文本中出现的扩展名），这是档 2 固有局限的机制原因。
- `edit` 权限规则的 `resource` 才是文件路径（作用于内置 `edit` / `write` / `patch` 工具），
  本方案不使用它；但正因 shell 通道不堵，任何按路径的精细控制都可能被 shell 绕过——
  这是三档都要保留 AGENTS.md 软指令的原因之一。
- 官方明确警告：黑名单式模式无法识别所有危险命令，应优先窄 allowlist。
  本方案 deny 的定位是"防呆 + 引导 AI 走内置工具"，**不是安全边界**。
- 全局指令文件：`~/.config/opencode/AGENTS.md`，自动注入所有会话。
  ⚠️ 一旦存在此文件，OpenCode 不再回退读取 `~/.claude/CLAUDE.md`，
  因此需把 `~/.claude/CLAUDE.md` 中仍需要的指令（如"使用中文回答"）迁移进 AGENTS.md。
- 配置保存后自动热重载，无需重启 OpenCode。

## 实施步骤（供代理执行）

### 0. 编码要求（必须先读）

中文 Windows 的 PowerShell 5.1 默认按 ANSI（GBK/CP936）读写无 BOM 文件：
`Set-Content` 写出的是 GBK，`Out-File` 写出的是 UTF-16LE，`Get-Content` 不加参数读 UTF-8 会乱码。
OpenCode 只按 UTF-8 读取 `opencode.json` 和 `AGENTS.md`，编码不对即乱码或解析失败。

因此强制以下规则：

1. **所有写文件操作必须使用内置 `edit` / `write` 工具**（恒为 UTF-8，无编码风险）。
   禁止用 `Set-Content` / `Out-File` / 重定向等 shell 方式创建或修改这两个文件——
   这既是本方案要拦截的行为，也必然损坏中文内容。本方案的 deny 规则与编码要求互为保障。
2. 读取 / 验证 JSON 时，`Get-Content` 一律加 `-Encoding UTF8`：
   `$env:USERPROFILE` 下的配置文件按 UTF-8 读，不要依赖默认编码。
3. 路径写法用 `$env:USERPROFILE\.config\opencode\...`，
   不要依赖 `~`（部分调用方式下不展开）。
4. 版本差异提示：PowerShell 7+ 全部默认 UTF-8（无 BOM），无此问题；
   Windows PowerShell 5.1（中文 Windows 系统自带）必须遵守上述规则。

### 1. 备份

若 `$env:USERPROFILE\.config\opencode\opencode.json` 已存在，
用内置工具（读取后写入副本的方式）在同一目录复制为 `opencode.json.bak-before-permissions`。
不要用 `Copy-Item` 之外的 shell 写文件方式改动原文件内容。
**每次档位切换前同样先做此备份**（覆盖旧备份即可）。

### 2. 写入 permissions（按用户所选档位）

通用守则（所有档位适用）：

- 只操作 `permissions` 数组，**严禁改动文件中其他字段**（provider、mcp、model 等）；
  文件不存在时新建，只含 `$schema` 和 `permissions`；
- 「本方案规则」的判定标准：与本 readme 列出的规则**字面完全一致**的条目
  （含首条 allow 基线）；
- 若现有数组含用户自有规则（与三档规则均不一致的条目），**保留它们**，
  并在执行前向用户说明哪些被保留；
- 本方案规则统一追加在数组末尾（后匹配优先，保证方案规则生效）；
- 切换档位 = 移除现有本方案规则 + 追加目标档位规则；
  卸载 = 仅移除本方案规则，数组变空则删除 `permissions` 键。

#### 档 3（strict）规则——18 条，与旧版单档方案完全一致

```json
[
  { "action": "*", "resource": "*", "effect": "allow" },
  { "action": "shell", "resource": "Set-Content *", "effect": "deny" },
  { "action": "shell", "resource": "set-content *", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *", "effect": "deny" },
  { "action": "shell", "resource": "add-content *", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *", "effect": "deny" },
  { "action": "shell", "resource": "out-file *", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *", "effect": "deny" },
  { "action": "shell", "resource": "clear-content *", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAllText*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAllLines*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAllBytes*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *", "effect": "deny" },
  { "action": "shell", "resource": "tee *", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *", "effect": "deny" },
  { "action": "shell", "resource": "tee-object *", "effect": "deny" },
  { "action": "shell", "resource": "*>*", "effect": "ask" }
]
```

#### 档 2（balanced）规则——136 条

结构：1 条 allow 基线 + 120 条 deny（15 扩展组 × 8 写方式，按组展开）+ 15 条 ask。
规则为纯 JSON（无注释），可整体复制进 `opencode.json` 或 `opencode.jsonc`：

```json
[
  { "action": "*", "resource": "*", "effect": "allow" },
  { "action": "shell", "resource": "Set-Content *.py*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.py*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.py*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.py*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.py*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.py*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.py*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.py*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.js*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.js*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.js*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.js*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.js*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.js*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.js*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.js*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.ts*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.c*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.c*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.c*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.c*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.c*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.c*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.c*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.c*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.h*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.h*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.h*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.h*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.h*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.h*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.h*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.h*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.go*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.go*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.go*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.go*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.go*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.go*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.go*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.go*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.rs*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.java*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.java*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.java*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.java*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.java*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.java*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.java*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.java*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.sh*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.ps1*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.sql*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.rb*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.php*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.php*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.php*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.php*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.php*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.php*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.php*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.php*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.swift*", "effect": "deny" },
  { "action": "shell", "resource": "Set-Content *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "Add-Content *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "Out-File *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "Clear-Content *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "Tee-Object *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "sed -i *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "*WriteAll* *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "*AppendAllText* *.kt*", "effect": "deny" },
  { "action": "shell", "resource": "*> *.py*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.js*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.ts*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.c*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.h*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.go*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.rs*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.java*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.sh*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.ps1*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.sql*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.rb*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.php*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.swift*", "effect": "ask" },
  { "action": "shell", "resource": "*> *.kt*", "effect": "ask" }
]
```

规则说明：首条 allow 保持其余操作默认放行；中间 deny 只拦「写方式 × 源码扩展名」；
末尾 ask 让"重定向写源码"需人工确认（`> build.log` 等非源码落点不匹配，放行）。

#### 档 1（soft）规则——1 条

```json
[
  { "action": "*", "resource": "*", "effect": "allow" }
]
```

### 3. 写入 AGENTS.md 标记块（按档位）

标记块格式（本方案的小节用标记包裹，**切换档位时只替换块内内容**，
AGENTS.md 其余内容——如语言条款——不在块内、一律不动）：

```md
<!-- oc-shell-deny:start mode=<strict|balanced|soft> -->
（对应档位的小节全文）
<!-- oc-shell-deny:end -->
```

操作规则：

- AGENTS.md 不存在时新建：写入头部（`# 全局指令`、`## 语言`、`- 使用中文回答。`）
  + 对应档位的标记块；
- 已存在且无本方案内容：把标记块**追加到末尾**，不覆盖原文；
- 已有本方案标记块：只替换块内小节与 mode 值；
- 检测到旧版部署（含"文件修改规则（强制）"但无标记块）：向用户确认后，
  把旧小节整体替换为标记块结构。

必须使用 `edit` / `write` 工具写入（保证 UTF-8，见步骤 0）。

#### 档 3（strict）标记块内容

```md
<!-- oc-shell-deny:start mode=strict -->
## 文件修改规则（强制）

1. 任何新建、修改、删除文件内容的操作，必须使用内置 `edit` / `write` 工具完成，禁止通过 shell 命令直接改写文件。
2. 禁止使用以下方式写文件（已被全局权限规则拦截）：
   - `Set-Content`、`Add-Content`、`Out-File`、`Clear-Content`、`Tee-Object`
   - `>` / `>>` 输出重定向写文件（如 `echo "x" > file.txt`）
   - `sed -i` 原地编辑
   - `[IO.File]::WriteAllText()` / `WriteAllLines()` / `AppendAllText()` / `WriteAllBytes()` 等 .NET API
3. shell 命令仅用于：构建、测试、运行程序、git 操作、包管理、只读查询（列目录、查看文件内容等）。
4. 确需通过 shell 写文件的例外场景（如生成构建日志），必须先向用户说明并征得同意。
5. `New-Item`、`Copy-Item`、`Move-Item` 等文件管理命令不得用于替代 `edit` / `write` 修改源代码文件内容。
<!-- oc-shell-deny:end -->
```

注：若该机器 `~/.claude/CLAUDE.md` 里有其他仍需生效的全局指令，一并迁移进 AGENTS.md（块外）。

#### 档 2（balanced）标记块内容

```md
<!-- oc-shell-deny:start mode=balanced -->
## 文件修改规则（源码保护 · 折衷档）

1. 修改、新建源代码文件必须使用内置 `edit` / `write` 工具。
   通过 shell 写源码文件（扩展名如 .py .js .ts .c .cpp .cs .css .h .html .go .rs .java .sh .ps1 .sql .rb .php .swift .kt 及其变体）会被权限规则硬拒绝；
   被拒后立即改用 `edit` / `write`，不要重试或变换 shell 写法。
2. 以下场景可用 shell，不受限制：写日志与构建输出（如 `npm run build > build.log`）、生成非源码产物、临时文件处理。
3. 重定向写源码文件会弹人工确认——确属必要再执行。
4. 禁止用变量间接方式绕过拦截（如 `$f = "a.py"; Set-Content $f`）；
   权限匹配基于命令文本，间接写法属于故意规避，一经发现视为违规。
5. `python -c` / `node -e` 等解释器一行式写文件不在拦截范围，
   但同样必须遵守第 1 条：源码一律使用 `edit` / `write`。
<!-- oc-shell-deny:end -->
```

#### 档 1（soft）标记块内容

```md
<!-- oc-shell-deny:start mode=soft -->
## 文件修改规则（源码保护 · 软性档）

1. 本档位无任何硬拦截：所有 shell 写文件操作都能成功执行且系统不会提醒，
   完全依赖你严格遵守以下规则。
2. 任何新建、修改、删除源代码文件内容的操作，必须使用内置 `edit` / `write` 工具。
3. 特别警告（中文 Windows）：Windows PowerShell 5.1 的 `Set-Content` 写出 GBK、
   `Out-File` 写出 UTF-16LE，写含中文的内容会产生编码损坏且**没有任何报错**，
   错误会在之后运行或打开文件时才暴露。内置工具恒为 UTF-8，无此风险。
4. 写日志、构建输出、非源码产物可用 shell。
5. 禁止 `sed -i` 原地编辑源码；禁止变量间接写源码。
<!-- oc-shell-deny:end -->
```

### 4. 验证（必须执行并汇报输出）

1. JSON 解析检查（强制 `-Encoding UTF8`）：

   ```powershell
   $c = Get-Content -Raw -Encoding UTF8 "$env:USERPROFILE\.config\opencode\opencode.json" | ConvertFrom-Json; $c.permissions.Count
   ```

   → 期望值按档位：档 3 = `18`、档 2 = `136`、档 1 = `1`（若保留了用户自有规则则更多）；
   卸载后应无 `permissions` 键或不含任何本方案规则。无 JSON 解析错误、无乱码。

2. 中文完整性抽查：

   ```powershell
   Get-Content -Encoding UTF8 "$env:USERPROFILE\.config\opencode\AGENTS.md" -TotalCount 10
   ```

   → 前几行的中文（"全局指令"、"使用中文回答"）应正常显示，不出现 `æ` / `锟斤拷` 之类乱码。
   若乱码，说明该文件不是 UTF-8（多半是哪一步用了 shell 写文件），按「常见问题」处理。

3. 档位标记检查（只读）：

   ```powershell
   Select-String -Path "$env:USERPROFILE\.config\opencode\AGENTS.md" -Pattern "oc-shell-deny:start"
   ```

   → 应显示 `mode=<strict|balanced|soft>` 且与所选档位一致；卸载后应无输出。

4. 行为测试（用代理的 **shell 工具**执行，测试文件都放 `$env:TEMP`，测完清理）。
   先建目录：`New-Item -ItemType Directory -Force -Path "$env:TEMP\oc-deny-test" | Out-Null`

   - **档 3**：`Set-Content -Path "$env:TEMP\oc-deny-test\a.txt" -Value test`
     → 应被拒（报 `Permission denied: shell`）。
   - **档 2**：
     - 拦截：`Set-Content -Path "$env:TEMP\oc-deny-test\app.py" -Value test` → 应被拒；
     - 放行：`Set-Content -Path "$env:TEMP\oc-deny-test\build.log" -Value test; Get-Content "$env:TEMP\oc-deny-test\build.log"`
       → 应成功且输出 `test`，无任何弹窗；
     - 确认：`"x" > "$env:TEMP\oc-deny-test\app.py"` → 应弹人工确认，选择 Reject 即可。
   - **档 1**：`Set-Content -Path "$env:TEMP\oc-deny-test\app.py" -Value test`
     → 应放行（证明无拦截）。
   - **卸载**：`Set-Content -Path "$env:TEMP\oc-deny-test\a.txt" -Value test` → 应放行。

   清理：`Remove-Item -Recurse -Force "$env:TEMP\oc-deny-test"`

   （若行为与档位不符：说明当前会话未热重载，提醒用户在 TUI 执行 `/reload` 或开新会话后再试。）

### 5. 汇报

列出：当前档位、修改的文件、追加/移除的规则数（及保留的用户自有规则数）、
四项验证的实际输出。

## 档位切换与卸载

用户可在任意时刻对代理说"切换到档 X"或"卸载限制"，代理按以下流程执行：

1. **识别当前状态**：读 AGENTS.md 的标记块 mode 值；
   无标记但含"文件修改规则（强制）"→ 旧版部署（按档 3 旧格式对待）；
   两者皆无 → 未部署/已卸载；
2. **确认目标状态**：展示四态菜单（含当前状态高亮），用户确认后才动手；
3. **执行**：按步骤 1 备份 → 按步骤 2 通用守则改 permissions
  （移除旧方案规则、追加目标档位规则；卸载则只移除）→
   按步骤 3 改 AGENTS.md 标记块（卸载则删除标记块）→
   按步骤 4 验证目标档位 → 按步骤 5 汇报。

## 回滚方式

用 `opencode.json.bak-before-permissions` 覆盖回去（恢复部署前配置）；
或直接切换到目标档位（保留方案、只调强度）。删除 AGENTS.md 中本方案的标记块
（若本次新建的整个文件则删除文件）。
恢复时同样只用 `edit` / `write` 工具，不要用 `Copy-Item` 覆盖配置文件内容。

## 常见问题

**三档怎么选？**
在意源码安全又想日常零打扰：档 2（推荐）；要最强编码防护、不想做任何判断：档 3；
信任模型遵守、追求极致效率：档 1。建议先从档 2 起步：被打扰太多降档 1，
在 git 历史里发现 shell 写源码的痕迹则升档 3。

**验证时 JSON 解析失败或中文乱码？**
定位是哪一步用了 shell 写文件（违反步骤 0）：从 `opencode.json.bak-before-permissions`
恢复原始内容，改用 `edit` / `write` 工具重做步骤 2、3。

**切换档位或新规则没有生效？**
配置虽热重载，但已开启的会话可能未刷新：在 TUI 执行 `/reload`，或开新会话再测。

**我自己加的 permissions 规则会丢吗？**
不会。代理只移除/追加与本 readme 规则**字面一致**的条目（"本方案规则"），
其余条目原样保留；执行前会向你列出保留了哪些。

**之前部署过旧版单档方案，如何迁移？**
代理会识别旧版特征（AGENTS.md 含"文件修改规则（强制）"但无 `oc-shell-deny` 标记、
permissions 含旧版 18 条规则），向你确认目标档位后按「档位切换与卸载」流程执行，
并补上标记块。旧版档 3 的规则与新版档 3 完全一致，无需改动。

**Windows PowerShell 5.1 和 PowerShell 7 有什么区别？**
PowerShell 7+ 全默认 UTF-8（无 BOM）；5.1 默认 ANSI（中文系统即 GBK）、`Out-File` 默认 UTF-16LE。
本方案（尤其档 3 与软指令文案）按最坏情况（5.1 + 中文环境）设计，两版都适用；
但任何版本下都应使用 `edit` / `write` 工具写文件，shell 只做只读验证。

**`~` 路径不展开？**
改用 `$env:USERPROFILE\.config\opencode\...` 的写法。
