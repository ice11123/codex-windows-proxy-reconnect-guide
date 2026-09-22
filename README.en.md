# Codex Reconnect Troubleshooting on Windows

**English** | [简体中文](./README.zh-CN.md)

> An evidence-first troubleshooting guide and reusable Agent instruction for repeated Codex `reconnect` / `retry` failures on Windows.

**Author: 离子怪**

## What this project solves

Codex Desktop or the Codex CLI may repeatedly reconnect. A bad proxy configuration is only one possible cause.

Authentication expiry, rate limits, upstream incidents, unstable proxy nodes, and third-party launchers can produce similar symptoms.

This repository provides an Agent guide that requires the Agent to:

1. inspect the Windows proxy and confirm the real listening port;
2. distinguish HTTP and SOCKS endpoints;
3. verify the network path with bounded requests;
4. apply recoverable user-level settings only when evidence supports the change;
5. restart affected processes safely and validate the new process environment;
6. stop guessing and collect precise errors when reconnects continue.

## Quick start

1. Download or open [`Codex-Windows-Proxy-Reconnect-Agent-Guide.md`](./Codex-Windows-Proxy-Reconnect-Agent-Guide.md).
2. Attach it to a working Codex task.
3. Describe the device, Codex client, reproduction time, and proxy application.
4. Ask the Agent to follow the document without guessing a port or modifying the system prematurely.

The original Chinese guide is available at [`Codex代理反复重连-Agent执行方案.md`](./Codex代理反复重连-Agent执行方案.md).

## Scope

- Windows 10 or 11;
- Codex Desktop or Codex CLI;
- a local HTTP proxy such as Clash Verge or Mihomo;
- PowerShell access to inspect the current user's proxy configuration.

## Out of scope

Proxy environment variables alone cannot fix:

- expired ChatGPT or API authentication;
- plan, balance, or rate-limit failures;
- OpenAI or relay-provider incidents;
- unstable or incompatible proxy nodes;
- interruptions caused by launchers, injection tools, or security software;
- enterprise certificate, inspection, or egress policies.

## Safety boundaries

> [!WARNING]
> This is an unofficial, community-maintained guide. Confirm the proxy address and protocol before making changes. Never guess a port, expose a credential-bearing proxy URL, or patch the Codex installation.

- Do not read, print, or commit API keys, tokens, passwords, or complete authentication objects.
- Do not treat `permissions.<name>.network.proxy_url` as an upstream proxy URL. It configures the local listener used by the Codex network sandbox.
- Do not assume Codex Desktop is guaranteed to load `~/.codex/.env`.
- Do not make the Agent terminate the Codex process serving its current task.
- Keep every change reversible and validate it in a newly started process.

## Official references

- [Codex environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
- [Codex advanced configuration](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex permissions and network proxy](https://learn.chatgpt.com/docs/permissions)

The official documentation describes `config.toml` as the persistent configuration entry point and distinguishes the Codex sandbox proxy listener from an upstream proxy.

General `HTTP_PROXY` variables are not listed as stable variables that Codex reads directly, so this project requires validation in a new process instead of treating a write as success.

## Repository contents

- [`README.md`](./README.md): bilingual landing page.
- [`README.zh-CN.md`](./README.zh-CN.md): Chinese project documentation.
- [`README.en.md`](./README.en.md): English project documentation.
- [`Codex代理反复重连-Agent执行方案.md`](./Codex代理反复重连-Agent执行方案.md): complete Chinese Agent guide.
- [`Codex-Windows-Proxy-Reconnect-Agent-Guide.md`](./Codex-Windows-Proxy-Reconnect-Agent-Guide.md): complete English Agent guide.
- [`LICENSE`](./LICENSE): MIT License.

## Related article

- [Codex reconnect troubleshooting: project introduction](https://ice11123.github.io/blog_test2/blog/ai-agent协作与开发/codex-windows-proxy-reconnect-guide/)

## Contributing

Issues and pull requests with results from other Codex versions, Windows environments, and proxy applications are welcome. Remove usernames, local paths, proxy credentials, tokens, and other secrets before sharing logs or screenshots.

## License

[MIT License](./LICENSE)
