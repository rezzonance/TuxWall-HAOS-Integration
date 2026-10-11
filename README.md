# TuxWall — Home Assistant integration

[![tuxwall.org](https://img.shields.io/badge/tuxwall.org-site-39ff14)](https://tuxwall.org/integrations.html)

Exposes a [TuxWall](https://tuxwall.org) gateway's security state as Home
Assistant entities and events, and closes the two blind spots neither side
can see alone: **east-west LAN traffic never touches the router**, so
attacks on Home Assistant itself are invisible to the gateway — and HAOS
knows its own auth state but has no gateway context. One event pipeline now
covers both.

The canonical source lives in the
[main TuxWall repository](https://github.com/rezzonance/tuxwall/tree/main/integrations/home-assistant/tuxwall);
this repository exists so HACS can install it with one click.

## Install (HACS)

1. HACS → ⋮ → **Custom repositories** → add
   `https://github.com/rezzonance/TuxWall-HAOS-Integration` with category
   **Integration**.
2. Find **TuxWall** in HACS → Download.
3. Restart Home Assistant, then **Settings → Devices & Services → Add
   Integration → TuxWall**.

Or manually: copy this repository's contents to
`/config/custom_components/tuxwall/` and restart Home Assistant.

## Setup

Create a **dedicated tuxwall user** on the gateway dashboard (Settings →
User Manager): viewer role is enough for sensors/events; admin only if you
want the ban/unban services. Use a strong password and **leave 2FA off for
this account** — TOTP codes rotate every 30 seconds and would break the
stored login. Keep 2FA on your human accounts.

Then enter the gateway's base URL (e.g. `http://192.168.1.1`) and those
credentials in the config flow. Poll interval is adjustable (10–3600s)
under the integration's Configure options.

## Entities

**Sensors** — attacks/min, firewall blocks, CrowdSec active bans, DNS
queries / blocked % / cache hit rate, WAN IP (interface + IPv6
attributes), VPN peers online, tarpit trapped connections.

**Binary sensors** — one per core gateway service (Unbound, Kea, radvd,
nginx, Suricata, CrowdSec + bouncer, WireGuard, the tuxwall units) plus
an aggregate "core services healthy" with a `down_services` attribute.

**Services** — `tuxwall.ban_device` / `tuxwall.unban_device` (UFW deny +
Unbound DNS block; requires an admin tuxwall user).

## Events

Fired as `tuxwall_event` with a `type`:

| type | when |
|---|---|
| `service_down` / `service_up` | a core gateway service transitions |
| `wan_ip_change` | the WAN address changed (dynamic WAN) |
| `tarpit_catch` | the honeypot trapped/auto-banned a source |
| `ha_login_failure` | HA's own failed-login alert (east-west blind spot) |

### Example automations

```yaml
alias: Gateway service down
trigger:
  - platform: event
    event_type: tuxwall_event
    event_data:
      type: service_down
action:
  - service: notify.mobile_app
    data:
      message: "TuxWall: {{ trigger.event.data.service }} stopped"
```

```yaml
alias: HA login failure
trigger:
  - platform: event
    event_type: tuxwall_event
    event_data:
      type: ha_login_failure
action:
  - service: notify.mobile_app
    data:
      message: "{{ trigger.event.data.message }}"
```

```yaml
alias: Lockdown when away
trigger:
  - platform: zone
    entity_id: person.jeff
    zone: zone.home
    event: leave
action:
  - service: tuxwall.ban_device
    data:
      ip: 192.168.1.99
```

## Security notes

- The gateway's dashboard API is **LAN/VPN-only** (the router's nginx
  denies WAN), so the integration connects from inside your network or
  over the router's WireGuard — by design.
- Keep the dedicated user at the lowest role that covers what you use.
- `ban_device` affects the whole LAN: double-check the IP before calling.

MIT License — see the
[main repository](https://github.com/rezzonance/tuxwall).
