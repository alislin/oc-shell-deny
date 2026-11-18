# opencode-shell-deny

为 OpenCode 配置全局限制：在所有项目中修改源代码文件时，只能使用内置 `edit` / `write` 工具，
禁止通过 shell 命令（PowerShell cmdlet、重定向、`sed -i` 等）直接改写文件。
实现方式 = 硬限制（permissions deny/ask）+ 软性指令（全局 AGENTS.md）。

## 适用范围

- 操作系统：Windows（含中文环境，PowerShell 编码问题见「编码要求」）
- Shell：PowerShell（Windows PowerShell 5.1 与 PowerShell 7 均适用）
- 配置对象：OpenCode 全局配置，对所有项目生效

## 如何使用（新主机部署）

把本仓库的 GitHub 链接发给代理（AI），并说明"按此文档完成配置"即可。
本文档是写给代理阅读并执行的说明书，**不提供也不需要任何安装脚本**。

代理职责：

1. 通读本文档全文，理解每一步之后再动手；
2. 阅读完成后，先向用户整理汇报"要做的事情"，内容包括：
   - 环境检测结果（`opencode.json` 是否已存在、是否已有 `permissions`、是否已有 AGENTS.md）
   - 将新建 / 修改哪些文件，追加多少条规则，备份文件名
   - 有无与现有配置冲突或需要用户决策的地方

   然后明确询问用户是否执行；**未经用户确认，不得开始任何改动**；
3. 遇到不确定的情况（已有配置冲突、异常报错、路径不存在等）先询问用户，不要擅自决定；
4. 按用户确认后的「实施步骤」顺序执行，不得跳步；
5. 完成后按「汇报」一节的要求输出结果。

## 它拦截什么

| 命令 / 模式 | 效果 | 说明 |
|---|---|---|
| `Set-Content` / `Add-Content` / `Out-File` / `Clear-Content` / `Tee-Object`（含小写变体） | deny | PowerShell 写文件 cmdlet |
| `[IO.File]::WriteAllText()` / `WriteAllLines()` / `AppendAllText()` / `WriteAllBytes()` | deny | 绕过 cmdlet 的 .NET 写文件 API |
| `sed -i` / `tee` / `Tee-Object`（含小写变体） | deny | 类 Unix 原地编辑、管道写入 |
| 任何含 `>` 输出重定向的命令 | ask | 无法区分"写日志"与"改源码"，交人工确认 |
| 其余所有操作 | allow | 构建、测试、git、包管理、只读查询不受影响 |

## 背景知识（官方机制，勿凭猜测）

- OpenCode 全局配置：`~/.config/opencode/opencode.json`
  （Windows 即 `C:\Users\<用户名>\.config\opencode\opencode.json`），对所有项目生效。
  全局 `permissions` 规则在项目规则之后追加、"最后匹配优先"，
  因此全局 deny 优先级高于任何项目级配置，且不可被 "Allow always" 审批覆盖。
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

### 2. 合并 permissions（不要覆盖已有配置）

读取现有 `opencode.json`，在顶层对象追加 `"permissions"` 键
（若已有 `permissions` 数组，则把下面规则追加到现有数组末尾）。
严禁改动或删除文件中已有的其他字段（provider、mcp、model 等）。
若文件不存在，新建文件只含 `$schema` 和 `permissions`：

```json
"permissions": [
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

规则说明：首条 allow 保持其余操作默认放行；中间 deny 拦截各类 shell 写文件命令
（大小写变体都要，因 PowerShell 大小写不敏感）；末条 ask 让任何带 `>`
输出重定向的命令需人工确认（无法区分"写日志"与"改源码"，故用 ask 不用 deny）。

### 3. 创建全局指令文件

创建 `$env:USERPROFILE\.config\opencode\AGENTS.md`
（若已存在，把下面小节追加到末尾，不要覆盖原文）。
必须使用 `edit` / `write` 工具写入（保证 UTF-8，见步骤 0）：

```md
# 全局指令

## 语言

- 使用中文回答。

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
```

注：若该机器 `~/.claude/CLAUDE.md` 里有其他仍需生效的全局指令，一并迁移进来。

### 4. 验证（必须执行并汇报输出）

1. JSON 解析检查（强制 `-Encoding UTF8`）：

   ```powershell
   $c = Get-Content -Raw -Encoding UTF8 "$env:USERPROFILE\.config\opencode\opencode.json" | ConvertFrom-Json; $c.permissions.Count
   ```

   → 应输出 `18`（若合并了已有规则则更多），且无 JSON 解析错误、无乱码。

2. 中文完整性抽查：

   ```powershell
   Get-Content -Encoding UTF8 "$env:USERPROFILE\.config\opencode\AGENTS.md" -TotalCount 10
   ```

   → 前几行的中文（"全局指令"、"使用中文回答"）应正常显示，不出现 `æ` / `锟斤拷` 之类乱码。
   若乱码，说明该文件不是 UTF-8（多半是哪一步用了 shell 写文件），按「常见问题」处理。

3. 拦截测试：用你的 **shell 工具**执行

   ```powershell
   Set-Content -Path "$env:TEMP\opencode-perm-test.txt" -Value test
   ```

   → 应被拒绝，报 `Permission denied: shell` 即为生效。
   （若未被拦截：说明当前会话未热重载，提醒用户在 TUI 执行 `/reload` 或开新会话后再试。）

### 5. 汇报

列出修改的文件、追加的规则数、三项验证的实际输出。

## 回滚方式

用 `opencode.json.bak-before-permissions` 覆盖回去；删除 `AGENTS.md`
中本次追加的小节（若本次新建的则整个删除）。
恢复时同样只用 `edit` / `write` 工具，不要用 `Copy-Item` 覆盖配置文件内容。

## 常见问题

**验证时 JSON 解析失败或中文乱码？**
定位是哪一步用了 shell 写文件（违反步骤 0）：从 `opencode.json.bak-before-permissions`
恢复原始内容，改用 `edit` / `write` 工具重做步骤 2、3。

**规则没有生效？**
配置虽热重载，但已开启的会话可能未刷新：在 TUI 执行 `/reload`，或开新会话再测。

**Windows PowerShell 5.1 和 PowerShell 7 有什么区别？**
PowerShell 7+ 全默认 UTF-8（无 BOM）；5.1 默认 ANSI（中文系统即 GBK）、`Out-File` 默认 UTF-16LE。
本方案的规则是按最坏情况（5.1 + 中文环境）设计的，两版都适用；但任何版本下都应使用
`edit` / `write` 工具写文件，shell 只做只读验证。

**`~` 路径不展开？**
改用 `$env:USERPROFILE\.config\opencode\...` 的写法。
