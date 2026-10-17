---
title: "Claude Code 接入指南"
permalink: /blogs/claude-code-access/
layout: single
author_profile: false
read_time: false
comments: false
share: false
related: false
sitemap: false
excerpt: "通过 Tailscale 和专属访问密钥，在自己的电脑上连接 Claude Code。"
---

> 这是一份提供给受邀用户的操作指南。**管理员会单独发给你两样东西：Tailscale 设备邀请链接、LiteLLM Virtual Key。** 请勿将邀请链接或密钥公开转发。
>
> 本教程不需要你自己部署服务器、购买 API 额度，也不需要把管理员的真实 API Key 放在自己的电脑上。

## 1. 你需要准备什么？

- 一台 macOS 或 Windows 电脑，并能够打开终端。
- 一个用于登录 [Tailscale](https://tailscale.com/) 的账号（可用自己的身份提供商账号注册）。
- 管理员发给你的 **Tailscale 设备邀请链接**。
- 管理员发给你的个人 **LiteLLM Virtual Key**（一般以 `sk-` 开头）。

**请区分两种凭证：** Tailscale 负责让电脑安全连接到网关；Virtual Key 负责识别你的 API 使用身份及预算。两者缺一不可。

## 2. 安装 Tailscale，接受设备邀请

1. 从 [Tailscale 官网](https://tailscale.com/download) 下载适合自己系统的客户端并安装。
2. 打开 Tailscale，登录自己的账号，确保状态为 **Connected**。
3. 打开管理员发来的**设备分享链接**，登录并接受邀请。
4. 在 Tailscale 的设备列表中确认能看到共享设备 **`lenovo-z475`**。

这是**共享单台服务器**，不是邀请你加入管理员的整个私人网络。之后使用 Claude Code 时，请保持 Tailscale 处于连接状态。

## 3. 检查网关连接

本教程使用的网关地址为：

```text
http://lenovo-z475.tail8387b0.ts.net:4000
```

其中 `lenovo-z475.tail8387b0.ts.net` 是 Tailscale MagicDNS 域名。共享设备应使用**完整域名**，而不是只写 `lenovo-z475`。

打开终端（Mac 使用 Terminal；Windows 使用 PowerShell），执行：

```bash
curl -i http://lenovo-z475.tail8387b0.ts.net:4000/health/readiness
```

如果看到 `HTTP/1.1 200 OK`，且响应包含以下内容，说明已连通：

```json
{"status":"healthy","db":"connected"}
```

> **Windows PowerShell 注意：** 某些旧版 PowerShell 把 `curl` 映射为 `Invoke-WebRequest`，可将命令中的 `curl` 改为 `curl.exe`。
>
> **域名无法解析？** 先确认已接受分享邀请、Tailscale 已连接且 MagicDNS 可用。必要时向管理员索取当前 Tailscale IP 作为备用；当前服务器的参考 IP 为 `100.77.28.11`，但共享网络中的实际地址可能不同。

## 4. 安装 Claude Code

### macOS

打开 Terminal，运行官方安装命令：

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

安装完成后，**重新打开一个终端窗口**，执行：

```bash
claude --version
```

### Windows

使用 PowerShell 运行官方安装命令：

```powershell
irm https://claude.ai/install.ps1 | iex
```

如果安装时提示缺少 Git，请先安装 [Git for Windows](https://git-scm.com/downloads/win)。完成安装后重新打开 PowerShell，并验证：

```powershell
claude --version
```

以上安装方式以 [Claude Code 官方仓库](https://github.com/anthropics/claude-code) 为准。如果安装界面和本教程不一致，优先参考官方最新说明。

## 5. 设置个人 Virtual Key 并启动

这里使用的是**你自己的 Virtual Key**，不是管理员的 Anthropic API Key。为避免留下明文命令历史，下方会通过交互式提示输入密钥。

### macOS（默认 zsh）

在**你准备运行 Claude Code 的终端窗口**里依次执行：

```bash
unset ANTHROPIC_API_KEY HTTPS_PROXY HTTP_PROXY ALL_PROXY
export ANTHROPIC_BASE_URL="http://lenovo-z475.tail8387b0.ts.net:4000"
export ANTHROPIC_MODEL="claude-opus-5-5"
read -rs "ANTHROPIC_AUTH_TOKEN?请粘贴你的 Virtual Key（输入不会显示）: "
echo
export ANTHROPIC_AUTH_TOKEN
claude --model claude-opus-5-5
```

粘贴 Virtual Key 后按回车。输入不会在屏幕上回显；启动后即可在 Claude Code 中提问。

### Windows（PowerShell）

在**你准备运行 Claude Code 的 PowerShell 窗口**执行：

```powershell
Remove-Item Env:ANTHROPIC_API_KEY -ErrorAction SilentlyContinue
Remove-Item Env:HTTPS_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:HTTP_PROXY -ErrorAction SilentlyContinue
$env:ANTHROPIC_BASE_URL = "http://lenovo-z475.tail8387b0.ts.net:4000"
$env:ANTHROPIC_MODEL = "claude-opus-5-5"
$secret = Read-Host "请输入你的 Virtual Key" -AsSecureString
$bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secret)
try {
    $env:ANTHROPIC_AUTH_TOKEN = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($bstr)
} finally {
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
}
claude --model claude-opus-5-5
```

以上环境变量**只在当前终端会话生效**。以后打开新窗口，需要重新设置。不要把 Virtual Key 写进公开仓库、共享截图或发送给其他人。

> 网关使用 `http://` 地址是有意设计：它只在 Tailscale 私人网络内开放，Tailscale 负责设备之间的加密传输。**不要关闭 Tailscale 后试图通过公网访问这个地址。**

## 6. 测试与日常使用

启动 Claude Code 后，可以先发送一句测试消息：

```text
Hello! Reply with one short sentence.
```

收到回复即可开始正常使用。建议在自己的项目目录中启动，例如 Mac：

```bash
cd ~/my-project
claude --model claude-opus-5-5
```

Windows PowerShell：

```powershell
cd C:\\path\\to\\my-project
claude --model claude-opus-5-5
```

日常使用记住两点：**先确保 Tailscale 已连接**；使用的终端里要有上述三个环境变量（`ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL`）。

你的 Virtual Key 可能有管理员设置的**累计美元预算**；即使密钥没有到期时间，预算用完也需要找管理员调整。

## 7. 常见问题

| 现象 | 建议检查 |
| --- | --- |
| `Could not resolve host` / 找不到域名 | Tailscale 是否已连接、邀请是否已接受；务必使用完整 `.ts.net` 域名 |
| `Connection timed out` | 先运行第 3 节的健康检查；联系管理员确认网关在线以及共享访问权限 |
| 健康检查 200，但 Claude Code 无法对话 | 检查当前终端的 `ANTHROPIC_BASE_URL`、模型名称和个人 Virtual Key |
| `401 Unauthorized` | 检查 Virtual Key 是否粘贴完整，是否误用了其他人的 Key |
| `403` 或预算相关错误 | 可能没有访问权限、额度用完或 Key 被停用；联系管理员 |
| `429` | 通常与限流或额度有关，稍后重试或联系管理员 |
| 新开终端后配置失效 | 本教程使用**会话级环境变量**，在新终端重新执行第 5 节 |
| 先前登录过个人 Claude 账号 | 先检查是否残留个人 API Key 或代理环境变量；本教程已提供清理命令 |

如果向管理员反馈问题，最好提供**报错文本、操作系统和发生时间**，但**不要发送完整 Virtual Key**。

---

*本文是受邀用户的快速接入指南。页面虽然不显示在本站博客列表中，但持有网址的人仍然可以访问；邀请链接和个人密钥请继续单独发送。*
