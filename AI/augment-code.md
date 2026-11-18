
## 安装
npm install -g @augmentcode/auggie@prerelease

auggie auth login
auggie auth status
auggie auth logout
auggie token print
auggie token revoke

index 目录：
auggie --workspace-root /path/to/your/project


忽略文件：
.gitignore 和 .augmentignore 中的文件会被忽略  
如果你的 .gitignore 文件中包含某个你希望被纳入索引的文件或目录，可以使用 ! 前缀将其添加到 .augmentignore 文件中。

禁用自动升级：
"autoUpdate": false in ~/.augment/settings.json
export AUGMENT_DISABLE_AUTO_UPDATE=1
AUGMENT_DISABLE_AUTO_UPDATE=1 auggie --print "Your instruction here"

auggie upgrade 手动升级

## 配置
`/settings`

配置文件从高到低优先级排序如下：
1. /etc/augment/settings.json 只读文件
2. <workspace>/.augment/settings.local.json 个人的项目配置文件，不应该提交到git
3. <workspace>/.augment/settings.json 共享的项目及配置文件，会作为源代码的一部分提交到 git
4. ~/.augment/settings.json 用户级的配置文件

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
Auggie 会自动从多个文件位置加载自定义规则和指南，以提供基于上下文的辅助功能。这些文件有助于 Auggie 理解您项目的约定、编码规范和偏好设置。
