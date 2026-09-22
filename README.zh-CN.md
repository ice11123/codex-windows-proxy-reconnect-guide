# Codex Windows 反复重连排障指南

[English](./README.en.md) | **简体中文**

> 一份先验证原因、再修改配置的 Codex Windows 代理排障说明，以及可直接交给 Agent 执行的完整 Markdown 指令。

**作者：离子怪**

## 解决什么问题

Codex 桌面端或 CLI 可能连续出现 `reconnect` / `retry`。代理配置错误只是可能原因之一，认证过期、服务端限流、上游故障、代理节点抖动和第三方启动器也可能产生相似现象。

本仓库提供一份可复用的 Agent 执行方案，要求 Agent：

1. 读取 Windows 系统代理并确认真实监听端口；
2. 区分 HTTP 与 SOCKS 代理，避免协议写错；
3. 使用有限超时的请求验证代理链路；
4. 仅在证据成立时写入可恢复的用户级代理设置；
5. 通过新进程和实际使用完成回归验证；
6. 失败时停止猜测，转而收集精确错误和日志。

## 快速使用

1. 下载或打开 [`Codex代理反复重连-Agent执行方案.md`](./Codex代理反复重连-Agent执行方案.md)。
2. 在当前能够正常工作的 Codex 任务中附上该文件。
3. 同时说明出现重连的设备、Codex 形态、复现时间和代理软件。
4. 要求 Agent 严格按文件执行诊断，不要预设端口或直接改系统。

如果只想自行排查，也可以逐节执行源文件中的只读检查、链路验证、配置、重启和回滚步骤。

## 适用范围

- Windows 10/11；
- Codex 桌面端或 Codex CLI；
- 本机已运行 HTTP 代理，例如 Clash Verge 或 Mihomo；
- 可以访问 PowerShell，并允许读取当前用户的代理配置。

## 不适用的情况

以下情况不能仅靠写入代理环境变量解决：

- ChatGPT 或 API 认证已经过期；
- 套餐、API 余额或速率限制导致请求失败；
- OpenAI 或中转服务上游故障；
- 代理节点本身不稳定或不支持所需连接；
- 第三方启动器、注入工具或安全软件中断连接；
- 企业网络使用了额外证书、审计或出口策略。

## 安全边界

> [!WARNING]
> 本项目是社区维护的非官方指南。执行前必须确认代理地址和协议，禁止猜测端口、泄露带凭据的代理 URL，或修改 Codex 安装包和 `app.asar`。

- 不读取、输出或提交 API Key、Token、密码和完整认证对象。
- 不把 `permissions.<name>.network.proxy_url` 当作上游代理地址；它用于 Codex 网络沙箱的本地代理监听。
- 不假设桌面端一定自动读取 `~/.codex/.env`。
- 不由正在服务当前任务的 Agent 强行结束 Codex。
- 所有修改都应能够回滚，并在新进程中重新验证。

## 官方依据

- [Codex 环境变量](https://learn.chatgpt.com/docs/config-file/environment-variables)
- [Codex 高级配置](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex 权限与网络代理](https://learn.chatgpt.com/docs/permissions)

官方文档将 `config.toml` 作为持久配置入口，并区分 Codex 网络沙箱监听代理与上游代理。`HTTP_PROXY` 等通用变量也不在 Codex 直接读取的稳定变量清单中，因此必须用新进程和实际请求验证。

本仓库不宣称 `~/.codex/.env` 是桌面端保证加载的公开接口。

## 仓库内容

- [`README.md`](./README.md)：双语入口。
- [`README.zh-CN.md`](./README.zh-CN.md)：中文项目说明。
- [`README.en.md`](./README.en.md)：英文项目说明。
- [`Codex代理反复重连-Agent执行方案.md`](./Codex代理反复重连-Agent执行方案.md)：中文完整 Agent 指令。
- [`Codex-Windows-Proxy-Reconnect-Agent-Guide.md`](./Codex-Windows-Proxy-Reconnect-Agent-Guide.md)：英文完整 Agent 指令。
- [`LICENSE`](./LICENSE)：MIT License。

## 延伸阅读

- [配套博客文章：Codex 反复重连怎么办](https://ice11123.github.io/blog_test2/blog/ai-agent协作与开发/codex-windows-proxy-reconnect-guide/)

## 贡献

欢迎提交 Issue 或 Pull Request，补充不同 Codex 版本、Windows 环境与代理软件的验证结果。提交日志或截图前，请移除用户名、本机路径、代理凭据、Token 和其他敏感信息。

## 作者与许可

作者：离子怪

本项目采用 [MIT License](./LICENSE)。
