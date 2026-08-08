# 06. Skills、系统提示词与上下文文件

## 1. 三者如何协作

Crush 给模型的系统提示由 Go template 生成。模板除了固定行为规范，还注入运行平台、日期、工作目录、Git 状态、用户上下文文件，以及可用 Skill 的**元数据目录**。Skill 正文不会全部塞入系统提示；模型先看到名称/描述/location，相关时再用 `view` 按需读取 `SKILL.md`。这样既省 Token，也能记录实际加载情况。

相关目录：

- `internal/agent/templates/coder.md.tpl`：主 coder Agent 行为。
- `task.md.tpl`：通用子 Agent，强调独立完成委托任务。
- `agentic_fetch_prompt.md.tpl`：网页研究子 Agent。
- `initialize.md.tpl`：初始化项目上下文文件。
- `title.md` / `summary.md`：标题和摘要的小模型提示。
- `agent_tool.md` / `agentic_fetch.md`：工具描述，不是系统提示。
- `internal/agent/prompt/prompt.go`：模板数据装配。
- `internal/skills/`：Skill 解析、发现、覆盖、目录与追踪。

## 2. Prompt 构造流程

`coderPrompt/taskPrompt/InitializePrompt` 只是把 `go:embed` 字节交给 `prompt.NewPrompt`。`Prompt.Build` 执行：

1. `text/template.Parse` 模板。
2. `promptData` 收集动态数据。
3. `template.Execute` 输出最终 system prompt。

`PromptDat` 字段包括 Provider、Model、完整 Config、WorkingDir、IsGitRepo、Platform、Date、GitStatus、本地/全局 ContextFiles、AvailSkillXML。

可通过 Option 注入时间、平台、workingDir，使测试可重复。WorkingDir 与 Platform 使用显式 Option 优先，否则用 ConfigStore 工作目录和 `runtime.GOOS`。

### 2.1 Git 上下文

只用 `workingDir/.git` 判断仓库。若是仓库，通过项目 Shell 运行：

- `git branch --show-current`
- `git status --short | head -20`
- `git log --oneline -n 3`

单项 git 命令失败被当作空信息而不是阻止 Agent 启动。路径最终转 slash，方便跨平台提示一致。

## 3. Context files

配置分别提供 `ContextPaths` 和 `GlobalContextPaths`。加载规则：

1. `~` 用 home helper 展开。
2. 以 `$` 开头的值交给 ConfigStore Resolver，支持配置变量/环境解析。
3. 使用 `SmartJoin(workingDir, path)`：相对路径相对项目。
4. 文件直接读取；目录用 `filepath.WalkDir` 递归读取全部非目录项。
5. 对展开后的路径做小写 key 去重。
6. 不可读文件跳过；目录遍历错误会结束该目录遍历。

Context 与 Global Context 在模板里是两个概念：项目规则通常属于前者，跨项目用户规则属于后者。它们是启动时直接注入正文，与 Skill 的按需加载不同。重新实现时要避免把敏感/大型目录无边界加入 context path。

## 4. SKILL.md 数据模型与格式

一个 Skill 是目录中的 `SKILL.md`，格式为 YAML frontmatter + Markdown body：

```markdown
---
name: my-skill
description: 何时使用以及它能帮助什么
user-invocable: true
disable-model-invocation: false
---

# Instructions
...
```

`Skill` 保存 name、description、user-invocable、disable-model-invocation、license、compatibility、metadata、正文 Instructions、目录 Path、文件 SkillFilePath、Builtin 标志。

解析器兼容 UTF-8 BOM、CRLF/CR，允许 frontmatter 前有空行，但要求 `---` 成对。校验规则：

- name 必填，最长 64；只允许字母数字和单连字符，不可首尾/连续连字符。
- name 必须与所在目录名大小写无关地一致。
- description 必填且最长 1024。
- compatibility 最长 500。

解析或校验失败不会停止整个发现，而是生成 `SkillState{StateError}` 供诊断/UI 展示。

## 5. Builtin Skills 与 `crush://`

`internal/skills/embed.go` 用 `//go:embed builtin/*` 把整个内置目录编进二进制。`DiscoverBuiltinWithStates` 遍历 embedded FS，解析校验后设置：

- `SkillFilePath = crush://skills/<name>/SKILL.md`
- `Path = crush://skills/<name>`
- `Builtin = true`

`crush://` 是虚拟只读协议，不是磁盘路径。View 工具识别 `skills.BuiltinPrefix`，从 `BuiltinFS()` 读 `builtin/<name>/SKILL.md`。Catalog 的 `ReadContent` 也走同样映射。

当前内置 Skill：

- `crush-config`：配置 Crush。
- `crush-hooks`：编写和排查 Hook。
- `jq`：jq 使用指南。

增加 builtin 只需创建 `internal/skills/builtin/<name>/SKILL.md`，embed glob 自动收录，并在 `TestDiscoverBuiltin` 添加断言。

## 6. 用户 Skill 发现、覆盖与禁用

标准生产入口是 `DiscoverFromConfig`：

1. 发现 builtin 与状态。
2. `ResolvePaths` 对 SkillsPaths 展开 `~` 和 `$VAR`。
3. `DiscoverWithStates` 用 fastwalk 并发遍历且跟随符号链接；同一文件 path 去重；结果按 path/name 排序，抵消并发遍历不确定性。
4. 发现顺序是 builtin 在前、用户 Skill 在后。
5. `Deduplicate` 按 name 保留**最后出现**，因此用户同名 Skill 覆盖 builtin。
6. `Filter` 按 `DisabledSkills` 精确名称移除，得到 activeSkills。
7. 状态也按 name last-wins，再按路径稳定排序。

注意三个集合：

- discovered：未去重原始集合。
- allSkills：已去重但包含 disabled，用于诊断“发现了但禁用”。
- activeSkills：去重且过滤禁用，用于提示、读取、追踪。

覆盖不是合并：用户同名 Skill 完全替代 builtin。Tracker 只允许 active name 被标记，因此读到已被覆盖的 builtin 路径也不会错误记为“有效 Skill 已加载”。

## 7. Manager：工作区隔离

`skills.Manager` 持有一个工作区的 all/active/states、已解析路径、workingDir 和独立 Broker。后端进程可同时托管多个 Workspace，不能共用 package global 状态。

`WithGlobalMirror` 只适用于单工作区进程（本地模式或客户端），它把 Manager 的状态同步到旧的 package-level cache/Broker。服务端多 Workspace 必须关闭 mirror。测试 `TestManager_ConcurrentWorkspacesAreIsolated` 固化此边界。

`PublishStates` 是状态变化的单一入口：先更新 Manager cache，再发布局部事件；启用 mirror 时再同步全局。返回状态使用深拷贝，防止调用者篡改缓存。

Coordinator 通常直接接收预发现 Manager 的快照。其内部 `discoverSkills` 只是遗留 fallback，不发布全局事件。

## 8. Skill 如何进入模型提示

`promptData` 当前会重新发现 builtin 和配置路径，发现用户覆盖时记录 warning，随后 Deduplicate + Filter，再调用 `ToPromptXML`：

```xml
<available_skills>
  <skill>
    <name>jq</name>
    <description>...</description>
    <location>crush://skills/jq/SKILL.md</location>
    <type>builtin</type>
  </skill>
</available_skills>
```

所有 XML 字段转义 `& < > " '`。`disable-model-invocation: true` 的 Skill 不出现在 XML，所以模型不会主动调用，但若 `user-invocable`，前端仍可把它作为用户命令展示/调用。这两个标志不要混淆：

- `user-invocable`：用户能否显式选择。
- `disable-model-invocation`：模型是否能从 available list 自主发现。

目录只含元数据，不含 Instructions。模型遵循 coder/task 模板里的规则，在任务匹配时调用 `view(location)` 读取全文。用户显式调用时可用 `FormatInvocation` 把名称、描述、location 和已转义正文包装成 `<loaded_skill>`。

## 9. Catalog、来源标签与读取

`Catalog(active, skillPaths, workingDir)` 面向前端产生 ID、Name、Description、Label、Source、UserInvocable。ID 就是 SkillFilePath。

来源判断：

- builtin → label `system:<name>`，source=system。
- 文件位于某 SkillsPath 下，且该 SkillsPath 又位于 workingDir 内 → project。
- 其他磁盘 Skill → user。

判断使用 `filepath.Rel` 并检查 `..`，防止简单字符串前缀误判。`ReadContent` 只允许读取 active 列表中精确匹配的 ID；disabled、被覆盖或任意传入路径都返回 `ErrSkillNotFound`，这是一层重要授权边界。

## 10. Tracker 与诊断

`Tracker` 建立时冻结 active name 集合，并发安全记录实际读取过的名称。View 成功读取 Skill 后调用 `MarkLoaded`；非 active name、nil tracker 都安全忽略。可查询排序后的 LoadedNames 和唯一数量。

Coordinator 每个 turn 前后对 LoadedNames 做差，并用 prompt 对 Skill 名/描述做廉价关键词相关性匹配，记录“新加载”和“看似相关但未加载”诊断。`crush_info` 展示 active、disabled、来源、loaded/unloaded 等状态。

粗略 Token 诊断使用 `(字节长度+3)/4`，只用于日志，不参与真实计费。

## 11. 模板各自职责

### 11.1 coder

主 Agent 的最高层工作协议：工具使用、编辑规范、路径与安全、上下文文件、Skill 发现方式、Git 状态等。模板根据 Config/Provider/Platform 条件生成不同片段。重新实现时应保持模板“描述行为”，而把权限、边界、并发强制放在代码中；不能只靠提示词保证安全。

### 11.2 task

给 `agent` 子 Agent。它得到自己的工作目录、上下文和可用 Skill，专注被委托任务；工具集合受 task Agent allowlist 限制，且不含交互 question。

### 11.3 agentic_fetch

工作目录是临时目录，不是项目目录；提示要求搜索、抓取、读取临时页面并返回证据化答案。工具集合在代码中硬限制，因此即使提示被网页内容诱导也不能直接编辑项目。

### 11.4 title 与 summary

它们不是完整 Agent 系统提示。Title 对首个真实 prompt 生成简短标题并剥离 think 标签。Summary 将历史与 todos 压缩成可继续工作的状态，必须保留目标、完成内容、未完成事项、关键文件与约束。

### 11.5 initialize

用于生成项目上下文指导文件，帮助新项目建立给 Agent 阅读的规则；它本身不参与每个 turn。

## 12. 测试所表达的不变量

- frontmatter、BOM、换行兼容；目录名与 name 校验严格。
- fastwalk 缺失路径不崩溃，合法结果稳定排序。
- user 同名项覆盖 builtin，disabled 在 dedup 后过滤。
- XML 正确转义，builtin 有 type，disable-model-invocation 被排除。
- Manager 默认不污染全局；mirror 明确开启才同步；多 Workspace 隔离。
- Catalog 只暴露 active；ReadContent 不能绕过 effective set。
- Tracker nil-safe、并发安全、只跟踪 active，覆盖后的 builtin 不会误记。
- View 能读取 `crush://`，并遵守大小限制。

## 13. 从零复刻顺序

1. 实现 `Skill`、frontmatter parser、Validate，并用表驱动测试覆盖坏格式。
2. 实现磁盘 DiscoverWithStates，确保 symlink、并发共享状态与稳定排序正确。
3. 实现 embedded builtin FS 和 `crush://` 读取。
4. 实现 Deduplicate(last-wins)、Filter 和 DiscoverFromConfig。
5. 实现 workspace Manager、Catalog/ReadContent 授权边界与 Tracker。
6. 实现 ContextPaths 路径展开、去重、目录递归。
7. 实现 PromptDat、Git 信息和模板渲染。
8. 最后将 Skill XML 注入模板、View 读取后 MarkLoaded、Coordinator turn 诊断连起来。

复刻完成的验收场景：创建一个用户 `jq` Skill 覆盖 builtin，并同时禁用另一个 Skill；系统提示只列有效项，`view` 读取用户 jq 后 tracker 显示 loaded，直接读取被覆盖的 `crush://skills/jq/SKILL.md` 不应被当作有效 jq 加载，两个并发 Workspace 的状态互不影响。
