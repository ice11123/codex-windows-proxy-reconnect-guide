# Codex 反复重连：Windows 代理配置 Agent 执行方案

> 目标：处理 Codex 经常出现重复 reconnect/retry（例如连续 5 次）的网络问题。必须先验证原因，不能看到重连就直接认定是代理故障。

本项目是社区维护的排障与执行指南，不是 OpenAI 官方项目。涉及 Codex 配置含义时，以文末链接的官方文档为准。

## 适用范围

- Windows 10/11 上的 Codex 桌面端或 Codex CLI。
- 本机已经运行 HTTP 代理，例如 Clash Verge / Mihomo。
- 代理地址形如 `http://127.0.0.1:端口`。

## 执行原则

1. 不猜代理端口。先读取系统代理并确认端口确实处于监听状态。
2. 不把 SOCKS 端口误写成 `http://`。
3. `~/.codex/.env` 可以作为统一记录，但不能假设 Codex 桌面端一定会自动加载它；Windows 上还应设置用户级环境变量。
4. 不修改 `WindowsApps`、Codex 安装包或 `app.asar`。
5. 不把 `permissions.<name>.network.proxy_url` 当成上游代理地址。它是 Codex 网络沙箱自身的本地监听配置。

## 第一步：发现真实代理

在 PowerShell 中只读检查：

```powershell
$settings = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'
$settings | Select-Object ProxyEnable, ProxyServer, AutoConfigURL

Get-NetTCPConnection -State Listen |
  Where-Object LocalAddress -in '127.0.0.1', '0.0.0.0', '::1', '::' |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
```

如果系统代理是 `127.0.0.1:7897`，则候选地址为：

```text
http://127.0.0.1:7897
```

如果没有明确地址、端口没有监听，停止修改并询问用户，不得自行编造端口。

## 第二步：构造可重复验证

将下面的占位符替换为已确认的真实地址：

```powershell
$proxyUrl = 'YOUR_CONFIRMED_HTTP_PROXY_URL'

curl.exe --silent --show-error --output NUL --write-out '%{http_code}' `
  --connect-timeout 5 --max-time 15 --proxy $proxyUrl https://chatgpt.com

curl.exe --silent --show-error --output NUL --write-out '%{http_code}' `
  --connect-timeout 5 --max-time 15 --proxy $proxyUrl https://api.openai.com/v1/models
```

判定：

- `chatgpt.com` 返回 `200`、`301`、`302` 或 `403`，说明 TLS 和代理链路已经建立。
- `/v1/models` 返回 `401` 也说明链路正常，只是该测试没有携带 API 凭据。
- `000`、连接超时或无法连接代理，才表示链路仍有问题。

## 第三步：写入 `~/.codex/.env`

使用文件编辑工具创建或更新 `%USERPROFILE%\.codex\.env`，内容如下。大小写版本都保留，便于不同子进程和工具读取：

```dotenv
HTTP_PROXY=YOUR_CONFIRMED_HTTP_PROXY_URL
HTTPS_PROXY=YOUR_CONFIRMED_HTTP_PROXY_URL
ALL_PROXY=YOUR_CONFIRMED_HTTP_PROXY_URL
NO_PROXY=localhost,127.0.0.1,::1

http_proxy=YOUR_CONFIRMED_HTTP_PROXY_URL
https_proxy=YOUR_CONFIRMED_HTTP_PROXY_URL
all_proxy=YOUR_CONFIRMED_HTTP_PROXY_URL
no_proxy=localhost,127.0.0.1,::1
```

注意：官方 Codex 文档将 `config.toml` 列为持久配置入口，并未承诺桌面端自动读取 `~/.codex/.env`。因此不能只创建这个文件就宣称修复完成。

## 第四步：设置 Windows 用户级代理环境变量

确认 `$proxyUrl` 已替换为真实地址后执行：

```powershell
$proxyUrl = 'YOUR_CONFIRMED_HTTP_PROXY_URL'
$noProxy = 'localhost,127.0.0.1,::1'

[Environment]::SetEnvironmentVariable('HTTP_PROXY',  $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('HTTPS_PROXY', $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('ALL_PROXY',   $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('NO_PROXY',    $noProxy,  'User')
```

Windows 环境变量名不区分大小写，因此用户环境中设置大写版本即可；`.env` 文件中可以同时保留大小写版本。

不要使用 `setx` 写入含敏感认证信息的代理 URL，因为命令行和日志可能泄露凭据。本地无认证回环代理通常不涉及这一问题。

## 第五步：让新配置真正生效

环境变量只会可靠地进入新启动的进程。要求用户：

1. 保存当前工作。
2. 完全退出 Codex，而不是只关闭窗口。
3. 如果使用 Dream Skin、Codex++ 或其他启动器，也完全退出对应托盘进程。
4. 重新登录 Windows；或者在广播环境变化后，从新的启动器进程重新打开 Codex。

不要由正在服务当前会话的 Agent 强行结束 Codex，否则会中断任务和对话。

## 第六步：回归验证

新会话中执行：

```powershell
$names = 'HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY', 'NO_PROXY'
foreach ($name in $names) {
  [PSCustomObject]@{
    Name = $name
    UserValue = [Environment]::GetEnvironmentVariable($name, 'User')
    ProcessValue = [Environment]::GetEnvironmentVariable($name, 'Process')
  }
}
```

要求：

- 四个 `UserValue` 均正确；
- 重启后的 Codex/Agent 中四个 `ProcessValue` 均正确；
- 代理 curl 测试成功；
- 实际使用中不再出现原先的连续重连。

若仍重连，必须收集重连发生时的精确错误、时间戳和日志，再区分 WebSocket 断开、服务端限流、认证过期、代理节点抖动、Dream Skin CDP 注入问题等原因，不能继续笼统归因于代理。

## 回滚

如需移除用户级设置：

```powershell
'HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY', 'NO_PROXY' | ForEach-Object {
  [Environment]::SetEnvironmentVariable($_, $null, 'User')
}
```

然后退出并重新启动 Codex；如不再需要，也可删除 `%USERPROFILE%\.codex\.env` 中对应行。

## 官方依据

- [Codex 官方环境变量文档](https://learn.chatgpt.com/docs/config-file/environment-variables)
- [Codex 官方高级配置文档](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex 官方权限与网络代理文档](https://learn.chatgpt.com/docs/permissions)

## 开源许可

Copyright (c) 2026 离子怪。

本项目采用 [MIT License](LICENSE)。
