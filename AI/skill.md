# skill
{docsify-updated}
> https://agentskills.io/home

一种为人工智能代理赋予新能力和专业知识的标准化方法。

## What are Agent Skills?
Agent Skills 是一种轻量级、开放的格式，用于通过专业知识和工作流来扩展人工智能代理的能力。

从本质上讲，一项技能是一个包含 `SKILL.md` 文件的文件夹。该文件包含元数据（至少包括 name 和 description）以及指导智能代理如何执行特定任务的操作说明。技能还可以打包脚本、参考资料、模板和其他资源。
```
my-skill/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

## Why Agent Skills?
Agent 的能力日益提升，但往往缺乏可靠完成实际工作所需的背景信息。Skills通过将流程知识以及针对公司、团队和用户的特定背景信息打包成可移植且受版本控制的文件夹，供 Agent 按需加载，从而解决了这一问题。这使 Agent 能够：
+ 领域专业知识：将专业知识（从法律审查流程到数据分析管道，再到演示文稿格式设置）转化为可重复使用的操作指南和资源。
+ 可重复的工作流：将多步骤任务转化为一致且可审计的流程。
+ 跨产品复用：只需构建一次技能，即可在任何兼容该技能的智能代理上使用。


## How do Agent Skills work?
Agent 通过渐进式披露分三个阶段加载技能：
1. 发现：在启动时，代理仅加载每个可用技能的名称和描述，这些信息仅够判断该技能何时可能相关。
2. 触发条件：当某项任务与某项技能的描述相符时，智能体将把完整的 `SKILL.md` 说明读入上下文中。
3. 执行：agent 按照指令操作，并可根据需要选择性地执行捆绑的代码或加载引用的文件。

完整指令仅在任务需要时才会加载，因此Agent可以在仅占用少量上下文空间的情况下，随时备有多种技能。


## Skill 规范
A skill is a directory containing, at minimum, a `SKILL.md` file:
```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

### SKILL.md 格式
`SKILL.md` 文件必须包含 `YAML` 前置信息，其后是 Markdown 内容。

```
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
metadata:
  author: example-org
  version: "1.0"
---
```

https://github.com/github/awesome-copilot  
https://awesome-copilot.github.com/
