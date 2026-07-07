# 安装学习笔记 Skill

将下面整段指令发送给兼容 Codex Skill 的节点：

```text
请使用 $skill-installer 安装 GitHub 仓库
https://github.com/dsxyy/dsxyy.github.io
中 `.agents/skills/publish-learning-notes` 目录下的 Skill。

执行要求：
1. 先读取 SKILL.md 和其直接引用的资源，确认目录符合 Codex Skill 规范。
2. 安装到当前用户的 Codex Skills 目录；优先使用当前环境提供的 Skill 安装工具。
3. 如果存在同名 Skill，先比较差异并向我确认，禁止直接覆盖。
4. 安装后执行格式校验，并报告安装路径、校验结果和一个整理触发示例、一个发布触发示例。
5. 只安装 Skill，不克隆或修改博客内容，不执行 git push。
```

安装完成后，可用以下指令验证触发：

```text
使用 $publish-learning-notes 把本次学习内容整理成博客草稿，但不要发布。
```
