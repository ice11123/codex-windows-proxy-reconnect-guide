# Codex Reconnects on Windows: Proxy Troubleshooting Agent Guide

> Author: 离子怪<br>
> Project: `codex-windows-proxy-reconnect-guide`

> Goal: diagnose repeated Codex `reconnect` / `retry` failures on Windows. Verify the cause before changing settings. A reconnect is not proof of a proxy failure.

## Important notice

This is an unofficial, community-maintained troubleshooting guide, not an OpenAI product. Follow the linked official documentation when interpreting Codex configuration.

The Agent must inspect the real environment before making changes. If the proxy address, protocol, or failure mode cannot be confirmed, stop and ask the user instead of inventing values.

## Scope

- Codex Desktop or Codex CLI on Windows 10 or 11.
- A local HTTP proxy is already running, such as Clash Verge or Mihomo.
- The proxy URL has the form `http://127.0.0.1:<PORT>`.

## Rules

1. Never guess the proxy port. Read the system proxy and confirm that the port is listening.
2. Never label a SOCKS endpoint as `http://`.
3. `~/.codex/.env` may record shared values, but do not assume Codex Desktop is guaranteed to load it. Also set Windows user-level variables when appropriate.
4. Do not modify `WindowsApps`, the Codex package, or `app.asar`.
5. Do not use `permissions.<name>.network.proxy_url` as the upstream proxy URL. It configures the local listener used by the Codex network sandbox.
6. Do not print or persist credentials embedded in a proxy URL.

## Step 1: discover the real proxy

Run these read-only checks in PowerShell:

```powershell
$settings = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'
$settings | Select-Object ProxyEnable, ProxyServer, AutoConfigURL

Get-NetTCPConnection -State Listen |
  Where-Object LocalAddress -in '127.0.0.1', '0.0.0.0', '::1', '::' |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
```

If the system proxy is `127.0.0.1:7897` and that port is listening as an HTTP or mixed proxy, the candidate URL is:

```text
http://127.0.0.1:7897
```

If no address is configured, the port is not listening, or only a SOCKS endpoint is available, stop. Report the evidence and ask the user for the correct HTTP proxy endpoint.

## Step 2: build a reproducible network check

Replace the placeholder with the confirmed URL:

```powershell
$proxyUrl = 'YOUR_CONFIRMED_HTTP_PROXY_URL'

curl.exe --silent --show-error --output NUL --write-out '%{http_code}' `
  --connect-timeout 5 --max-time 15 --proxy $proxyUrl https://chatgpt.com

curl.exe --silent --show-error --output NUL --write-out '%{http_code}' `
  --connect-timeout 5 --max-time 15 --proxy $proxyUrl https://api.openai.com/v1/models
```

Interpret the result:

- `chatgpt.com` returning `200`, `301`, `302`, or `403` shows that the proxy and TLS path were established.
- `/v1/models` returning `401` also proves network reachability; the request simply did not include API credentials.
- `000`, a connection timeout, or failure to connect to the proxy indicates a remaining network-path problem.

Do not treat an HTTP authentication or authorization response as a transport failure.

## Step 3: record values in `~/.codex/.env`

Create or update `%USERPROFILE%\.codex\.env` with the confirmed URL. Keep upper- and lower-case forms for tools that read different conventions:

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

The official Codex documentation describes `config.toml` as the persistent configuration entry point. It does not promise that Codex Desktop automatically loads `~/.codex/.env`, so creating this file alone is not proof of a fix.

## Step 4: set Windows user-level proxy variables

After replacing the placeholder with the verified URL, run:

```powershell
$proxyUrl = 'YOUR_CONFIRMED_HTTP_PROXY_URL'
$noProxy = 'localhost,127.0.0.1,::1'

[Environment]::SetEnvironmentVariable('HTTP_PROXY',  $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('HTTPS_PROXY', $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('ALL_PROXY',   $proxyUrl, 'User')
[Environment]::SetEnvironmentVariable('NO_PROXY',    $noProxy,  'User')
```

Windows environment variable names are case-insensitive, so setting the upper-case user values is sufficient. The `.env` file may keep both forms for cross-tool compatibility.

`HTTP_PROXY`, `HTTPS_PROXY`, and `ALL_PROXY` are general process proxy variables. They are not listed among the stable environment variables that Codex documents as reading directly.

The permissions documentation only states that an active Codex network sandbox proxy can respect these upstream settings. Writing the variables is therefore a configuration step to verify, not proof that Codex adopted the proxy.

Avoid `setx` for proxy URLs containing credentials because command histories and logs may expose them. A credential-free loopback proxy normally does not have this risk.

## Step 5: make the new environment effective

Environment changes reliably reach newly started processes, not processes that are already running. Tell the user to:

1. save current work;
2. fully quit Codex, not only close a window;
3. fully quit any launcher or tray process such as Dream Skin or Codex++;
4. sign in to Windows again, or broadcast the environment change and start Codex from a new launcher process.

The Agent serving the current task must not forcibly terminate Codex. Doing so would interrupt the task and conversation.

## Step 6: regression validation

In a newly started task, inspect both user and process values:

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

Completion requires:

- all four `UserValue` entries are correct;
- the restarted Codex or Agent has the expected `ProcessValue` entries;
- the bounded proxy requests succeed;
- the original repeated reconnect symptom no longer reproduces during normal use.

If reconnects continue, collect the exact error, timestamp, and relevant logs. Distinguish WebSocket disconnects, server rate limits, expired authentication, node instability, and third-party launcher interference.

Do not continue to blame the proxy without new evidence.

## Rollback

Remove the Windows user-level values with:

```powershell
'HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY', 'NO_PROXY' | ForEach-Object {
  [Environment]::SetEnvironmentVariable($_, $null, 'User')
}
```

Then fully quit and restart Codex. If the `.env` entries are no longer needed, remove only those proxy lines from `%USERPROFILE%\.codex\.env`.

## Required final report

Report observed values and results without exposing secrets:

```text
Codex reconnect diagnosis completed:

- Codex client: Desktop / CLI
- System proxy enabled: yes / no
- Confirmed proxy protocol: HTTP / mixed / SOCKS / unknown
- Confirmed listener: <HOST:PORT or not found>
- Bounded network check: passed / failed
- User environment updated: yes / no
- New process environment verified: yes / no
- Reconnect reproduction: resolved / still present / not yet retested
- Rollback available: yes
- Remaining evidence needed: <none or exact items>
```

Do not report success solely because a file or environment variable was written.

## Official references

- [Codex environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
- [Codex advanced configuration](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex permissions and network proxy](https://learn.chatgpt.com/docs/permissions)

## License

Copyright (c) 2026 离子怪.

This guide is released under the [MIT License](./LICENSE).
