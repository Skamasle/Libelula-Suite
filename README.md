# Libelula Postfix Modules

Libelula Postfix Modules is a Postfix-focused suite built to solve two common operational problems in outbound mail infrastructures:

- centralized runtime rate limit management
- automatic IP rotation and blacklist-aware transport selection across relay clusters, including Proxmox Mail Gateway clusters

It provides Redis-driven runtime services, a lightweight PHP control panel, and a clean split between persistent configuration and live operational state.

## Short Project Description

Redis-driven Postfix modules for centralized rate limiting, outbound IP rotation, temporary penalties, and a lightweight PHP admin panel for multi-server relay clusters.

## Main Goals

- Provide centralized rate limit management for Postfix environments.
- Provide automatic outbound IP rotation for relay clusters.
- Exclude blacklisted IPs dynamically without editing each server by hand.
- Keep Postfix hot paths independent from MySQL.
- Preserve a Redis-driven runtime model.
- Keep the panel focused on operational control rather than unrelated mail platform administration.

## Compatibility

Libelula supports:

- standard Postfix deployments
- multi-node relay platforms
- Proxmox Mail Gateway environments

It has been tested in Proxmox Mail Gateway environments through Postfix-compatible integration patterns.

## Production Testing

The platform is in production testing.

The ratelimit service has been operating with a memory footprint in the range of 8 MB to 32 MB of RAM at peak usage, with 40k daily mails.

## Components

### `libelula-ratelimit`

A Go-based Postfix policy service.

- reads compiled runtime configuration from Redis
- evaluates live counters stored in Redis
- applies limits atomically via Redis/Lua
- supports SPF-related checks and SPF Redis cache
- supports temporary multipliers read from Redis
- keeps runtime config separate from static file config

### `libelula-selector`

A lightweight local service for outbound transport selection.

- reads a local pool of transports or IPs
- consumes blacklist state from Redis
- excludes blacklisted IPs from the active pool
- updates active transport selection without requiring a Postfix restart
- returns a healthy local transport to Postfix
- uses a fallback route or IP if all node-local options become unavailable

### `libelula-punisher`

A small CLI for temporary runtime penalties.

- writes multiplier keys into Redis
- increases or decreases an existing penalty
- inspects current multiplier state
- deletes temporary penalties

It stays decoupled from the ratelimit engine and only affects runtime behavior while the multiplier exists.

### PHP Admin Panel

The PHP panel provides:

- admin login
- user creation
- persistent ratelimit rules
- ratelimit overrides
- runtime publication into Redis
- ratelimit status visibility from Redis
- manual selector blacklist management

## Architecture

Libelula keeps persistent configuration and live runtime state deliberately separated.

### MySQL stores persistent configuration

Examples:

- panel users
- panel settings
- ratelimit global rules
- ratelimit overrides
- selector blacklist metadata
- other editable operational settings

### Redis stores live runtime state

Examples:

- current compiled ratelimit config
- current ratelimit config version
- live counters
- temporary multiplier keys
- selector blacklist set
- SPF cache

## Runtime Control Flow

The intended control-plane pattern is:

1. Operators edit configuration in the PHP panel.
2. The panel stores persistent values in MySQL.
3. The panel compiles a runtime JSON view.
4. The panel publishes the runtime view into Redis.
5. Go services consume Redis at runtime.
6. Go services do not query MySQL in the SMTP hot path.

Redis is not optional in the final architecture. It is the shared runtime backend.

## Cluster-Oriented Design

Libelula supports multi-server environments.

Clustered model:

- multiple Postfix nodes
- one shared Redis runtime backend
- one shared MySQL configuration backend
- one selector instance per node
- node-local transport pools
- shared blacklist consumption across the cluster

Operational assumptions:

- many ratelimit workers may write to the same Redis counter space
- many selector instances may read the same blacklist set
- each node may keep its own local pool and fallback route
- the panel remains the configuration authority
- Redis remains the shared runtime authority

Use this model for:

- outbound relay farms
- VPS-based sending clusters
- distributed abuse-control systems
- Postfix clusters with shared traffic controls

## Postfix Compatibility

The project is based on standard Postfix concepts:

- policy services for SMTP decisions
- local sockets or local TCP listeners
- transport selection helpers
- normal relay and cleanup flows

Run `libelula-ratelimit` as a Postfix policy daemon.

Common integration points:

- `check_policy_service`
- `smtpd_recipient_restrictions`
- `smtpd_end_of_data_restrictions`

Use `libelula-selector` as a local decision layer that gives Postfix a healthy outbound transport based on current blacklist state.

The product uses generic Postfix behavior and works in Proxmox Mail Gateway environments through Postfix-compatible flows.

## Postfix Integration Notes

The selector requires Postfix transport entries in `master.cf` for each outbound IP or relay identity that may be selected at runtime.

### Example `master.cf` entries for `libelula-selector`

```cf
relay1-10.0.10.170 unix  -  -  n  -  -  smtp
  -o smtp_bind_address=10.0.10.170
  -o smtp_helo_name=relay1-170.skamasle.com
  -o syslog_name=postfix
  -o smtp_reply_filter=pcre:/etc/postfix/tag_10_0_10_170.pcre
  -o smtp_tls_security_level=may

relay1-10.0.10.171 unix  -  -  n  -  -  smtp
  -o smtp_bind_address=10.0.10.171
  -o smtp_helo_name=relay1-171.skamasle.com
  -o syslog_name=postfix
  -o smtp_reply_filter=pcre:/etc/postfix/tag_10_0_10_171.pcre
  -o smtp_tls_security_level=may
```

Each transport name must match the naming model consumed by `libelula-selector`, for example:

- `relay1-10.0.10.170`
- `relay1-10.0.10.171`

`libelula-selector` can then return one of those transport names dynamically based on the current active pool and Redis blacklist state.

### Local transport pool file on each node

For `libelula-selector` to work correctly, each node needs a local file containing the transports or IPs that belong to that node.

Keep that file at:

```text
/opt/libelula/etc/transport_pool
```

Each entry must include the node prefix so `libelula-selector` can return valid Postfix transport names.

Example contents for `relay1`:

```text
relay1-10.0.10.170
relay1-10.0.10.171
relay1-10.0.10.172
```

The same file can also contain plain IPs if `libelula-selector` is configured to derive the final transport name from the node name. In production, fully expanded transport names are usually clearer.

This file is node-local. In other words:

- each relay node keeps its own transport pool file
- the file only contains transports or IPs that belong to that node
- the node prefix must match the transport naming defined in `master.cf`
- `libelula-selector` filters that local pool against the shared Redis blacklist without requiring a Postfix restart when a blacklisted IP is removed from active selection

### Example `main.cf` for `libelula-selector`

To let Postfix ask `libelula-selector` for the transport decision:

```cf
sender_dependent_default_transport_maps = socketmap:unix:/run/libelula/libelula-selector.sock:selector
smtp_connection_cache_on_demand = no
```

If your deployment keeps the socket in another path, adjust it accordingly.

### Relay restrictions for `libelula-ratelimit`

For relay services that apply Libelula policy checks, configure relay restrictions at the service level in `master.cf`.

Example:

```cf
-o smtpd_relay_restrictions=permit_mynetworks,reject_unauth_destination
-o smtpd_recipient_restrictions=$libelula_policy_check
```

### Example `main.cf` policy definition

Define the Postfix policy service in `main.cf`:

```cf
libelula_policy_check = check_policy_service { inet:127.0.0.1:10040, timeout=3s, default_action=DUNNO }
```

This points Postfix to `libelula-ratelimit`, which can then enforce runtime rate limits using the compiled Redis configuration and live Redis counters.

## systemd Services

Deploy the runtime services with systemd:

- `libelula-ratelimit.service`
- `libelula-selector.service`

Repository files:

- `systemd/libelula-ratelimit.service`
- `systemd/libelula-selector.service`

Typical operations:

```bash
systemctl daemon-reload
systemctl enable libelula-ratelimit.service
systemctl enable libelula-selector.service
systemctl start libelula-ratelimit.service
systemctl start libelula-selector.service
systemctl status libelula-ratelimit.service
systemctl status libelula-selector.service
```

Typical log inspection:

```bash
journalctl -u libelula-ratelimit.service -f
journalctl -u libelula-selector.service -f
```

`libelula-ratelimit.service` runs the Postfix policy daemon and serves policy requests on the configured local listener.

`libelula-selector.service` runs the local selector daemon, reads the node-local transport pool, consumes the shared Redis blacklist, and updates active transport selection without requiring a Postfix restart.

## Ratelimit Scopes

Libelula supports several runtime scopes. Each scope targets a different abuse pattern.

### `client_ip`

This is the base brake by sender VPS. It limits the total volume a client IP can send in each time window.

### `sender`

Applies the rate limit directly to the full `MAIL FROM`, for example `hola@dominio.com`, even if it changes VPS or destination.

### `sender_size`

Groups by `sender + size`, for example `hola@dominio.com|20519`, to detect bursts of repeated messages with the same sender and message size.

### `client_ip_sender_domain`

Controls campaigns concentrated on the same `MAIL FROM` domain from a specific VPS. It helps detect abuse even when the VPS total volume is not extreme yet.

### `client_ip_recipient_prefix2_domain`

This is the per-VPS dictionary rate limit. It groups by `IP|first 2 recipient letters + domain`, for example `192.168.10.2|ro+hotmail.com`, and helps stop enumeration or brute force attacks against similar mailboxes.

### `sender_recipient_prefix2`

Groups by `sender + first 2 recipient letters`, for example `mail@listgac.com|aa`, to slow dictionary-style campaigns sent from the same sender.

## SPF and Advanced Runtime Options

### Redis TTL (s)

Redis cache time for a domain already checked for SPF. It avoids repeating the same DNS query on every message while the TTL is valid.

### DNS Timeout (ms)

Maximum wait time for the DNS server to answer the sender domain TXT/SPF query.

### Multiplier engine

Allows temporarily lowering the effective limit of an IP, sender, or sender domain by reading multipliers from Redis. It does not change base rules or overrides. It only adjusts the final effective limit while the penalty exists.

### Empty MAIL FROM

If a message is rejected due to SPF and the client forged the sender, for example `hola@google.com`, some VPS try to send a bounce afterwards to `hola@google.com` with an empty sender. This option blocks that bounce to avoid outgoing backscatter.

## Redis Model

Current default runtime keys in this repository follow a `libelula:*` naming style.

Examples:

- `libelula:ratelimit:config:current`
- `libelula:ratelimit:config:version`
- `libelula:ratelimit:counter:*`
- `libelula:ratelimit:multiplier:*`
- `libelula:ratelimit:spf:*`
- `libelula:selector:blacklist`

## Selector and Blacklist Flow

1. Blacklist state exists outside `libelula-selector`.
2. An external checker, operator action, or orchestration layer publishes blacklist state into Redis.
3. `libelula-selector` periodically refreshes its local in-memory pool.
4. Blacklisted IPs disappear from active routing without editing Postfix on each request.
5. A fallback route or IP may be used if all node-local candidates are excluded.

`libelula-selector` consumes blacklist state. Detection can stay outside `libelula-selector`.

In clustered environments, all nodes react to shared blacklist updates through Redis instead of requiring manual changes on every server.

One practical model used in real environments is a Nagios-compatible `check_rbl` workflow. In that model, the RBL checker does not make routing decisions itself. Instead, it detects blacklisted outbound IPs and publishes that state so `libelula-selector` can remove those IPs from the active relay pool.

## External Blacklist Producers

Blacklist detection stays modular.

Valid producers:

- an external reputation checker
- a cron task
- a Nagios-compatible `check_rbl` workflow
- manual panel actions

In a Nagios-style model, the flow is:

1. `check_rbl` checks the reputation or blacklist status of outbound IPs.
2. The checker or an orchestration layer writes the active blacklist set into Redis.
3. `libelula-selector` refreshes its local cache from Redis.
4. Blacklisted IPs are excluded from active rotation automatically.
5. Postfix keeps receiving only healthy transports from `libelula-selector`.

Responsibilities:

- `check_rbl` is a detector
- Redis is the shared runtime state
- `libelula-selector` is the consumer that applies routing exclusion

Flow:

1. A checker reviews outbound IP reputation.
2. The checker writes or rebuilds a blacklist set in Redis.
3. `libelula-selector` refreshes from Redis periodically.
4. `libelula-selector` excludes bad IPs from the active pool.
5. Postfix receives only healthy transports.

## Repository Layout

```text
cmd/
  libelula-ratelimit/
  libelula-selector/
  libelula-punisher/
configs/
docs/
schema/
panel/
scripts/
```

## Screenshots

### Ratelimit Configuration

![Libelula Ratelimit Configuration](https://skamasle.com/media/posts/23/responsive/ratelimit-config-2-2xl.png)

### Ratelimit Status

![Libelula Ratelimit Status](https://skamasle.com/media/posts/23/responsive/ratelimit-status-2xl.png)

## Panel

The repository includes a PHP panel with:

- login
- bootstrap admin creation on first login if no users exist
- user management
- ratelimit configuration
- ratelimit runtime publication to Redis
- ratelimit status inspection from Redis
- manual selector blacklist management

## Bootstrap Credentials

If the `panel_users` table is empty, the panel can bootstrap the first admin login with:

- username: `admin`
- password: `admin123!`

These values can be overridden with:

- `LIBELULA_BOOTSTRAP_USER`
- `LIBELULA_BOOTSTRAP_PASS`

Change them immediately in production.

## Environment Variables

The PHP panel reads configuration from environment variables.

### MySQL

- `LIBELULA_DB_DSN`
- `LIBELULA_DB_USER`
- `LIBELULA_DB_PASS`

### Redis

- `LIBELULA_REDIS_HOST`
- `LIBELULA_REDIS_PORT`
- `LIBELULA_REDIS_PASSWORD`
- `LIBELULA_REDIS_TIMEOUT`
- `LIBELULA_RL_CONFIG_KEY`
- `LIBELULA_RL_VERSION_KEY`
- `LIBELULA_RL_COUNTER_PREFIX`
- `LIBELULA_SELECTOR_BLACKLIST_KEY`

## Database Migration

The SQL schema file is:

- `libelula_panel.sql`

It creates the tables needed for:

- panel users
- panel settings
- ratelimit global rules
- ratelimit overrides
- selector manual blacklist entries

## Build Notes

Each Go component is an independent module.

```bash
cd cmd/libelula-ratelimit
go test ./...

cd ../libelula-selector
go test ./...

cd ../libelula-punisher
go test ./...
```

## License

GNU GPL v3. See `LICENSE`.

## Credits

Created by Maksim Usmanov.
