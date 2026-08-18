---
name: mihoro
description: >
  Manage the mihoro/mihomo proxy setup (Mihomo core + mihoro CLI + mihoro.omarchy bar widget).
  Use when editing ~/.config/mihoro.toml or ~/.config/mihomo/config.yaml, restarting or
  debugging mihomo.service, enabling/disabling TUN mode, switching proxy modes, updating
  the subscription, or troubleshooting why proxy/TUN traffic fails. Triggers: mihoro,
  mihomo, clash, proxy, VPN, TUN, subscription, 7890, 9090, omarchy-mihoro plugin.
  On this machine GitHub is unreachable; all GitHub downloads go through the gh-proxy.com
  mirror (see Mirror section).
---

# Mihoro / Mihomo Proxy Skill

Manages the proxy stack on this Omarchy machine: the `mihomo` core daemon, the
`mihoro` CLI that drives it, and the `mihoro.omarchy` bar widget.

## Architecture

| Component | Role | Location |
|-----------|------|----------|
| **mihomo** | Clash/Mihomo core daemon | `~/.local/bin/mihomo`, config dir `~/.config/mihomo/` |
| **mihoro** | CLI: init/apply/update/status, mode switching | `~/.local/bin/mihoro` |
| **mihoro.toml** | mihoro's config + overrides applied on top of mihomo config | `~/.config/mihoro.toml` |
| **mihomo config.yaml** | The live mihomo config (proxies, groups, rules, tun, dns) | `~/.config/mihomo/config.yaml` |
| **mihomo.service** | systemd user service | `systemctl --user mihomo.service` |
| **mihoro.omarchy** | Omarchy bar widget (status, mode switch, subscription) | `~/.config/omarchy/plugins/mihoro.omarchy/` |

Key ports/endpoints:

- mixed proxy port: `7890` (also `7891` HTTP / `7892` SOCKS)
- REST API / controller: `http://127.0.0.1:9090/` (dashboard at `/ui/`)
- TUN interface: `Meta` (fake-ip range `198.18.0.1/16`)

## Key commands

```bash
mihoro status                # service + API status
mihoro apply                 # apply [mihomo_config] overrides from mihoro.toml, restart service
mihoro update                # refetch subscription config (config -> geodata -> core -> ui)
mihoro update --all          # everything, then restart
mihoro proxy export          # eval $(mihoro proxy export) — set proxy env for a shell
mihoro proxy unset           # clear proxy env
mihoro cron enable/disable   # auto-update cron
systemctl --user restart mihomo.service
```

## How config is merged

`mihoro` treats `~/.config/mihomo/config.yaml` as remote config and applies the
`[mihomo_config]` section of `~/.config/mihoro.toml` as overrides. Overridden
fields: `port, socks-port, mixed-port, redir-port, allow-lan, bind-address,
mode, log-level, ipv6, external-controller, external-ui, secret, geodata-mode,
geo-auto-update, geo-update-interval, geox-url`.

Unknown keys (e.g. `tun`, `dns`) survive `mihoro apply` untouched because the
parser flattens extras. To change those, edit `config.yaml` directly and run
`mihoro apply` (or restart the service).

If you write a full subscription YAML by hand into `config.yaml`, keep the
overrides in `mihoro.toml` consistent with it (e.g. `allow_lan`, `external_controller`).

## Mirror (CRITICAL on this machine)

GitHub is unreachable from this network (raw.githubusercontent.com and
github.com both time out). Everything GitHub-hosted must go through a mirror:

- A global git rewrite is configured: `github.com` -> `https://gh-proxy.com/https://github.com/`
  in `~/.gitconfig`. `git clone https://github.com/...` works through it.
- mihoro runtime downloads (core/geodata) need the env var:
  `MIHORO_GITHUB_MIRROR="https://gh-proxy.com" mihoro update` (mihoro appends
  the full GitHub URL to this prefix).
- To remove the git rewrite:
  `git config --global --unset url."https://gh-proxy.com/https://github.com/".insteadOf`

## TUN mode

Enabled via the `tun:` block in `~/.config/mihomo/config.yaml`.

**MUST use `stack: gvisor`.** `system` and `mixed` stacks create the `Meta`
interface and even answer ping, but silently drop TCP (all connections time
out; nothing appears in `/connections`). `gvisor` is the only stack verified
working on this machine.

```yaml
tun:
  enable: true
  stack: gvisor        # DO NOT use system/mixed — TCP breaks
  dns-hijack:
  - any:53
  auto-route: true
  auto-detect-interface: true
```

DNS uses fake-ip mode:

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  default-nameserver: [223.5.5.5, 119.29.29.29]
  nameserver: [https://doh.pub/dns-query, https://dns.alidns.com/dns-query]
  nameserver-policy: { "geosite:cn": [223.5.5.5, 223.6.6.6] }
```

Prerequisites (already done on this machine):

- TUN capabilities on the mihomo binary:
  `sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service=+ep ~/.local/bin/mihomo`
- Verify with `getcap ~/.local/bin/mihomo` (should list `cap_net_admin,cap_net_raw,cap_net_bind_service=ep`).
- Toggle TUN: set `tun.enable` true/false in `config.yaml`, then `mihoro apply`.

Diagnosing TUN:

- `ip link show Meta` — interface should exist and be `UP`.
- `getent hosts www.google.com` — should return `198.18.x.x` (fake-ip = DNS hijack working).
- `ip route get 1.1.1.1` — should resolve via `dev Meta table 2022`.
- `curl https://www.gstatic.com/generate_204` — 204 means full TUN routing works.
- If TCP hangs but ping to a fake-ip replies: wrong `tun.stack`, switch to `gvisor`.
- mihomo logs: `journalctl --user -u mihomo.service -n 30 --no-pager`.

## omarchy bar widget

- Plugin id: `mihoro.omarchy` (enabled, bar section right).
- Reads `external_controller` and `secret` from `mihoro.toml`'s `[mihomo_config]`;
  the plugin substitutes wildcard hosts to `127.0.0.1`.
- Manage: `omarchy plugin list | rg mihoro`, `omarchy plugin update mihoro.omarchy`,
  `omarchy plugin remove mihoro.omarchy`.

## Subscription updates

The subscription URL lives in `remote_config_url` in `~/.config/mihoro.toml`.
`mihoro update` refetches it. If it's still the placeholder
(`https://example.com/subscription`) the update will fail — set the real URL first.
After updating, the file at `~/.config/mihomo/config.yaml` is re-merged with
overrides; re-check `tun`/`dns` sections survive if they matter (the update
re-downloads the whole remote config).

## Troubleshooting

- **Service fails to start / immediate exit**: `journalctl --user -u mihomo.service -n 30 --no-pager`.
- **Config syntax error**: mihomo prints it in the journal; check the matching
  section in `config.yaml`, fix, `mihoro apply`.
- **No proxies / placeholder subscription**: edit `remote_config_url` then `mihoro update`.
- **TUN up but connections time out**: stack not `gvisor` (see above), or
  capabilities missing (`getcap ~/.local/bin/mihomo`).
- **API unreachable**: confirm `external-controller` in config and that the
  plugin's controller address matches `~/.config/mihoro.toml`.
- **GitHub clone fails**: the git insteadOf rewrite should handle it; if removed,
  re-add it or pass the mirror URL explicitly.
