
## 安装
`npm install -g @augmentcode/auggie@prerelease`

```
auggie auth login
auggie auth status
auggie auth logout
auggie token print
auggie token revoke
```

index 目录：
`auggie --workspace-root /path/to/your/project`


忽略文件：
`.gitignore `和 `.augmentignore` 中的文件会被忽略  
如果你的 `.gitignore` 文件中包含某个你希望被纳入索引的文件或目录，可以使用 `!` 前缀将其添加到 `.augmentignore` 文件中。

禁用自动升级：
`"autoUpdate": false` in `~/.augment/settings.json`
`export AUGMENT_DISABLE_AUTO_UPDATE=1`
`AUGMENT_DISABLE_AUTO_UPDATE=1 auggie --print "Your instruction here"`

`auggie upgrade` 手动升级

## 配置
`/settings`

配置文件从高到低优先级排序如下：
1. `/etc/augment/settings.json` 只读文件
2. `<workspace>/.augment/settings.local.json` 个人的项目配置文件，不应该提交到git
3. `<workspace>/.augment/settings.json` 共享的项目及配置文件，会作为源代码的一部分提交到 git
4. `~/.augment/settings.json` 用户级的配置文件

## subagents
子代理是可自定义的代理，您可以对其进行配置以执行特定任务，例如代码审查或项目初始化。每个子代理都拥有自己的工具和功能，您可以将其用于处理特定任务。您可以使用本地子代理，也可以通过将配置添加到代码库中与团队共享这些子代理。
+ 子代理拥有独立于主代理的上下文窗口
+ 子代理具有自定义提示，用于向代理下达指令
+ 子代理与其他子代理并行运行
+ 子代理将在主线程中显示其当前进度摘要

Auggie 包含三个内置代理：`general-purpose` 、 `explore` 和 `plan` 。如果创建了同名的自定义代理，则该自定义代理将覆盖内置代理。

可以使用 `/agents` 命令创建代理，也可以使用配置文件创建。


可以通过创建配置文件手动创建子代理。该配置文件应为 Markdown 文件，并存储在 `~/.augment/agents/`（仅限用户）或 `./.augment/agents/`（共享）目录下。如：
```
---
name: code-review
description: Code review agent
model: sonnet
color: purple
---

You are an agentic code-review AI assistant with access to the developer's codebase through Augment's deep codebase context engine and integrations. You are conducting a comprehensive code review for the staged changes in the current working directory.

## Review Areas to focus on:

- **Potential Bugs**: Identify bugs, logic errors, edge cases, crash-causing problems.
- **Security Concerns**: Look for potential vulnerabilities, input validation, authentication issues ONLY if the code is security-sensitive
- **Documentation**: Report comments or documentation that is incorrect or inconsistent with the code.
- **API contract violations**
- **Database and schema errors**
- **High Value Typos**: typos that affect correctness, UX-strings, etc.
```

配置好子代理后，可以通过发送一条引用该代理名称的消息来触发它。Augment 还会自动检测任务是否适合由子代理处理，并建议使用该子代理。
```
> Use the code-review agent to review my staged changes
```

更多详细配置参考： https://docs.augmentcode.com/cli/subagents


## rules
Agent and Chat rules and guidelines are natural language instructions that can help Augment reply with more accurate and relevant responses. Rules and guidelines are perfect for telling Augment to take into consideration specific preferences, package versions, styles, and other implementation details that can’t be managed with a linter or compiler. You can create guidelines for a specific workspace or globally for all chats in your IDE/Agent.

Auggie 会自动从多个文件位置加载自定义规则和指南，以提供基于上下文的辅助功能。这些文件有助于 Auggie 理解您项目的约定、编码规范和偏好设置。

+ Custom rules (via the --rule flag): a file or directory
+ Workspace rules folder: `<workspace_root>/.augment/rules/` - Recursively searches `.md` and `.mdc` files
+ User rules folder: `~/.augment/rules/` - Recursively searches `.md` and `.mdc` files for user-wide rules
+ `AGENTS.md` or `CLAUDE.md`: Loaded from the current directory and every parent directory. In each directory, Auggie loads `AGENTS.md` if it exists and otherwise `CLAUDE.md`
+ Workspace guidelines: `<workspace_root>/.augment-guidelines`

以交互模式运行 `/rules` 命令，以查看已加载的规则及其类型。使用 `--no-rules` 选项可关闭规则检测功能。

### System prompt files
还有两个文件会更改系统提示符本身：
+ `.augment/APPEND_SYSTEM.md` or `~/.augment/APPEND_SYSTEM.md` : 追加 Auggie 的默认系统提示符后边
+ `.augment/SYSTEM.md` or `~/.augment/SYSTEM.md` : 替换 Auggie 的默认系统提示符

### Rules File Format
规则文件应采用 Markdown 格式编写，并包含自然语言说明。以下是推荐的结构：
```markdown
# Project Guidelines

## Code Style
- Use TypeScript for all new JavaScript files
- Follow the existing naming conventions in the codebase
- Add JSDoc comments for all public functions and classes

## Architecture
- Follow the MVC pattern established in the codebase
- Place business logic in service classes
- Keep controllers thin and focused on request/response handling

## Testing
- Write unit tests for all new functions
- Maintain test coverage above 80%
- Use Jest for testing framework

## Dependencies
- Prefer built-in Node.js modules when possible
- Use npm for package management
- Pin exact versions in package.json for production dependencies
```

### Hierarchical Rules
```
my-project/
  AGENTS.md                  <- Always included (workspace root)
  src/
    AGENTS.md                <- Included when working in src/ or subdirectories
    frontend/
      AGENTS.md              <- Included when working in src/frontend/
      App.tsx
    backend/
      AGENTS.md              <- Included when working in src/backend/
      server.ts
  tests/
    AGENTS.md                <- Only included when working in tests/
```
在处理 `src/frontend/App.tsx` 时：
+ 加载了 `src/frontend/AGENTS.md`（当前目录）
+ 加载了 `src/AGENTS.md`（父目录）
+ 通过标准规则加载了工作区根目录下的 `AGENTS.md`
+ 未加载 `src/backend/AGENTS.md` 和 `tests/AGENTS.md`（位于不同分支）


### Important Notes
+ 仅会按层次结构查找 `AGENTS.md` 和 `CLAUDE.md` 文件。如果某个目录中同时存在这两个文件，则仅使用 `AGENTS.md`
+ `.augment/rules/` 目录下的文件仅从工作区根目录加载，不从子目录加载
+ 规则按对话会话进行缓存，以避免重复包含
+ 层次结构规则会与工作区规则和用户规则相结合

### Best Practices for Rules Files
+ **Be Specific**: Provide clear, actionable guidelines rather than vague suggestions
+ **Use Examples**: Include code examples when describing patterns or conventions
+ **Keep Updated**: Regularly review and update rules as your project evolves
+ **Be Concise**: Focus on the most important guidelines to avoid overwhelming the AI
+ **Test Guidelines**: Verify that Auggie follows your rules by testing with sample requests


## skills
技能在位于 `.augment/skills/` 、 `.claude/skills/` 或  `.agents/skills/` 目录（位于您的工作区或主目录中）的 `SKILL.md` 文件中进行定义。每个技能必须位于其独立的子目录中：
```
.augment/skills/
  ├── python-testing/
  │   └── SKILL.md
  ├── api-design/
  │   └── SKILL.md
  └── database-migrations/
      └── SKILL.md
```

### Skill Locations
技能是从以下位置获取的，按优先级顺序排列：
| Location | Source | Description |
| --- | --- | --- |
| `~/.augment/skills/` | User | Available in all workspaces (highest precedence) |
| `<workspace>/.augment/skills/` | Workspace | Project-specific skills, can be version controlled |
| `~/.claude/skills/` | User | Compatible with Claude Code |
| `~/.agents/skills/` | User | Industry-standard location |
| `.augment/skills/`, `.claude/skills/`, `.agents/skills/` in the current directory and its parents | Workspace | Searched up to the git repository root |

来自所有位置的技能都会被加载，并提供给客服代表使用。当存在多个同名技能时，系统将使用优先级较高的位置中的技能。

还可以加载其他路径中的技能：
+ 使用 `--skill <路径>` 参数加载单次运行所需的技能文件或目录。重复使用该参数可添加更多技能。
+ 在 `settings.json` 文件中向 `skills` 数组添加路径。
+ 使用 `--no-skills` 参数关闭自动发现功能。通过 `--skill` 参数传递的技能仍会加载。

### Invoking Skills
在交互模式下，每个发现的技能都会被注册为一个名为 `/skill:<名称>` 的斜杠命令。如果你在 `.augment/skills/my-skill/SKILL.md` 路径下有一个技能，请输入以下命令来运行它：
```
/skill:my-skill
```` 
命令后面的任何文本都会连同其说明一起传递给该技能：
```
/skill:my-skill focus on the payments module
```