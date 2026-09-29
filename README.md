# 单文件 HTML 工具开发与验证

> WorkBuddy Skill · 屋里涛说

交付「一个 .html 双击即用、零依赖、不联网」的工具——批量裁剪/重命名/转换、数据看板、表单等，`file://` 协议下运行，所有逻辑内嵌单文件。

## 技能清单

| 技能 | 说明 |
|---|---|
| **web-tool-verify** · HTML 工具开发 | 开发要点 + 浏览器级验证套路，保证交付即可用。 |


## 安装

把 `skills/` 下的技能目录拷贝到 WorkBuddy 的技能目录：

```bash
cp -r skills/* ~/.workbuddy/skills/
```

Windows PowerShell：

```powershell
Copy-Item .\skills\* "$env:USERPROFILE\.workbuddy\skills\" -Recurse -Force
```

重启 WorkBuddy 后，技能列表即可看到。

## 使用要点

- 核心约束：零依赖、离线可用、双击即开、单文件交付。

## 环境依赖

- 浏览器（验证）

## 目录规范

```
web-tool-verify/
└── skills/
    ├── web-tool-verify/
```

每个技能遵循统一结构：`SKILL.md`（必需，含 name/description frontmatter）+ `scripts/`（可选）+ `references/`（可选）。

---

## License

MIT — 随意取用、修改、二次分发。
