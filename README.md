# Codex Figma Plugins

这是 Stone Team 用于通过 GitHub Marketplace 向团队分发 Codex Plugin 的私有仓库。当前包含 `figma-structure` 插件，用于对 Figma 界面、图层、Frame、Auto Layout、组件和变量做结构化分析与处理。

本仓库只负责分发 Plugin 和 Skill。安装插件不会自动获得任何 Figma 文件权限。每位团队成员仍需使用自己的 Figma 账号单独连接并完成授权。

## 目录结构

```text
codex-figma-plugins/
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── plugins/
│   └── figma-structure/
│       ├── plugin.json
│       └── skills/
│           └── figma-structure/
│               ├── SKILL.md
│               ├── references/
│               ├── scripts/
│               ├── assets/
│               ├── agents/
│               └── config/
├── .gitignore
└── README.md
```

## 团队安装方法

仓库是 private。安装前请确认 GitHub 账号已被加入 `stone-team-plugins/codex-figma-plugins`，并且本机 GitHub CLI 已登录该账号。

```bash
codex plugin marketplace add stone-team-plugins/codex-figma-plugins --ref main
codex plugin marketplace list
```

执行后还需要：

1. 重新启动 ChatGPT/Codex 桌面客户端。
2. 打开 Plugins Directory。
3. 选择 Stone Team Plugins。
4. 找到 Figma Structure。
5. 点击 Install。
6. 单独完成 Figma 登录和授权。

## Figma 授权说明

- GitHub 仓库只分发 Plugin 和 Skill，不内置 Figma Access Token、OAuth Secret 或文件权限。
- 安装 Plugin 不会自动获得 Figma 文件权限。
- 团队成员必须分别登录自己的 Figma 账号并完成授权。
- 未授权时，Skill 无法读取或修改 Figma 文件。
- 本仓库不包含 MCP 服务器密钥。请在本机 Codex / ChatGPT 中连接并授权 Figma。

## 更新方法

Skill 或 Plugin 更新并推送到 `main` 后，团队成员执行：

```bash
codex plugin marketplace upgrade stone-team-plugins
```

然后重新启动 ChatGPT/Codex 桌面客户端，再打开 Plugins Directory 确认 `Figma Structure` 版本已更新。

## 私有仓库权限要求

- 仓库：`https://github.com/stone-team-plugins/codex-figma-plugins`
- 权限：private
- 团队成员至少需要对该仓库的 read 权限，才能添加 Marketplace 并安装插件。
- 没有仓库访问权限时，`codex plugin marketplace add` 会失败。

## 常见问题

**Marketplace 列表里看不到 Stone Team Plugins？**  
确认已经执行 `codex plugin marketplace add stone-team-plugins/codex-figma-plugins --ref main`，并且当前 GitHub 账号能访问该私有仓库。然后重启客户端。

**能看到 Marketplace，但找不到 Figma Structure？**  
打开 Plugins Directory，选择 Stone Team Plugins，确认插件名称是 Figma Structure。

**安装成功但无法读取 Figma？**  
这通常不是插件安装问题。请单独完成 Figma 登录和授权。安装 Plugin 不会自动授予 Figma 文件权限。

**更新后还是旧版本？**  
先执行 `codex plugin marketplace upgrade stone-team-plugins`，再重启客户端。必要时新开一个线程后再使用。
