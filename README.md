# Codex Windows Proxy Reconnect Guide

## Codex 反复重连：Windows 代理排障与 Agent 执行方案

[English](./README.en.md) | [简体中文](./README.zh-CN.md)

An evidence-first troubleshooting guide and reusable Agent instruction for repeated Codex `reconnect` / `retry` failures on Windows.

用于诊断 Codex 在 Windows 上反复 `reconnect` / `retry` 的开源排障指南与可直接交给 Agent 执行的 Markdown 指令。

## Choose a language / 选择语言

- [English documentation](./README.en.md)
- [简体中文说明](./README.zh-CN.md)

## Agent guides / Agent 执行指令

- [English Agent guide](./Codex-Windows-Proxy-Reconnect-Agent-Guide.md)
- [中文 Agent 执行方案](./Codex代理反复重连-Agent执行方案.md)

The guide requires diagnosis before configuration changes. It checks the real proxy listener, verifies the network path, applies recoverable user-level settings, restarts affected processes safely, and validates the result.

本方案坚持先取证、后修改：确认真实代理监听与协议，验证网络链路，再写入可回滚的用户级设置，并在新进程中完成回归验证。

> [!WARNING]
> This is an unofficial, community-maintained troubleshooting guide. Reconnects can also be caused by authentication, rate limits, upstream outages, unstable nodes, or third-party launchers.
>
> 这是社区维护的非官方排障方案。重连也可能由认证、限流、上游故障、节点抖动或第三方启动器引起，不能看到重连就直接认定是代理故障。

**Author / 作者：离子怪**<br>
**License / 许可证：[MIT](./LICENSE)**
