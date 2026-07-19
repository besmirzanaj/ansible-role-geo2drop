# nftables geo2drop — Design Spec

## Overview

Replace the firewalld+ipset backend in `ansible-role-geo2drop` with native nftables
sets, and create a standalone `besmirzanaj/geo2drop` fork providing the same
nftables-based geo-blocking script independently of Ansible.

Both repos maintain their own lifecycle with minimal coupling: the fork is a
self-contained bash script, the Ansible role templates its own version of the
script and adds deployment, scheduling, and idempotent configuration management.

## Nftables Table Design

### Ruleset

```
table inet geo2drop {
    set blcountries {
        type ipv4_addr
        flags interval
        auto-merge
    }
    set blcountries6 {
        type ipv6_addr
        flags interval
        auto-merge
    }

    chain geo2drop_input {
        type filter hook input priority -10; policy accept;
        ip saddr @blcountries counter drop
        ip6 saddr @blcountries6 counter drop
    }

    chain geo2drop_forward {
        type filter hook forward priority -10; policy accept;
        ip saddr @blcountries counter drop
        ip6 saddr @blcountries6 counter drop
    }
}
```

### Design decisions

| Decision | Rationale |
|---|---|
| `inet` family | Single table covering both address families |
| `priority -10` | Runs before firewalld (priority 0) and most custom rules. Only blocked packets are dropped; the rest flow through to other tables unchanged. |
| `policy accept` | Unmatched packets are not dropped by this table. Geo-blocking adds drops, not a default-deny. |
| `auto-merge` | nftables merges adjacent CIDRs (e.g. `10.0.0.0/24` + `10.0.1.0/24` → `10.0.0.0/23`), keeping the set compact. |
| `counter` on drop rules | Provides visibility via `nft list table inet geo2drop` without separate monitoring. |
| Separate v4/v6 sets | ipdeny serves different URLs for IPv4 and IPv6 zone files. Two sets avoid mixing address families. |
| input + forward hooks | Covers traffic destined to the host and traffic routed through it (matching firewalld's `drop` zone). |

### Coexistence with existing rules

The script never flushes other tables, never deletes rules outside the `geo2drop`
table, and uses `nft add element` (not `nft flush set`) for updates. Existing
firewalld rules, nftables rulesets, and iptables remain untouched.

## Standalone Script — `besmirzanaj/geo2drop`

### CLI interface

```
geo2drop [--config PATH] [--force] [--dry-run]
```

- `--config PATH` — override config path (default `/etc/geo2drop/geo2drop.conf`)
- `--force` — rebuild the nftables set even if the entry hash is unchanged
- `--dry-run` — print what would be done without modifying nftables

### Script flow

1. Read config file (shell variables, sourced)
2. Validate dependencies: `nft`, `curl`, `tar`, `sha256sum`, `sort`, `grep`
3. Ensure nftables service is active
4. Fetch zone data from ipdeny (online/archive/local modes, same as current)
5. Build sorted, deduplicated entry list from country zone files
6. Filter out whitelisted CIDRs from the entry list
7. Compute sha256 of the filtered entry list
8. If hash differs from `applied.sha256` (or `--force`):
   - Create table + chains + sets (idempotent — `nft add *` is a no-op if exists)
   - Add drop rules (idempotent — `nft add rule` with `position` or replace existing)
   - Populate the set with `nft add element inet geo2drop blcountries { ... }`
   - Save new hash
   - Print `CHANGED`
9. If unchanged: ensure the set exists and drop rules are present, print `UNCHANGED`

### Config variables

```
COUNTRIES="br id cl"
WHITELIST="203.0.113.0/24 198.51.100.0/24"
SET_NAME="blcountries"
SET_NAME6="blcountries6"
TABLE_FAMILY="inet"
SOURCE_MODE="online"
IPDENY_URL="https://www.ipdeny.com"
GITHUB_ARCHIVE_URL="https://github.com/..."
ARCHIVE_FALLBACK_GITHUB="true"
DOWNLOAD_TIMEOUT="30"
STATE_DIR="/var/lib/geo2drop"
ZONES_DIR="${STATE_DIR}/zones"
ARCHIVE_DIR="${STATE_DIR}/archive"
```

### Whitelist mechanism

The `WHITELIST` variable contains space-separated CIDRs. The script filters
these out of the desired entries *before* populating the nftables set. If a
whitelisted CIDR overlaps with a country zone, it is excluded from the set.

This avoids adding a separate nftables rule for each whitelist entry — the
entries simply never make it into the set.

### Output

```
[geo2drop] countries: br id
[geo2drop] source=online: fetching country zones
[geo2drop] entry set unchanged (hash=abc123): not touching nftables
UNCHANGED
```

Or on change:

```
[geo2drop] countries: br id
[geo2drop] source=online: fetching country zones
[geo2drop] entry set changed (old=abc123 new=def456): rebuilding
[geo2drop] creating table inet geo2drop (already exists, skipping)
[geo2drop] populating set blcountries with 4281 entries
[geo2drop] whitelist filtered 3 entries
CHANGED
```

## Ansible Role Changes

### `vars/main.yml`

```yaml
geo2drop_packages:
  - nftables
  - curl
  - tar
  - gzip
```

### `defaults/main.yml`

```yaml
geo2drop_countries:
  - br
  - id
  - cl
geo2drop_whitelist: []
geo2drop_set_name: blcountries
geo2drop_nft_family: inet
geo2drop_manage_nftables: true
geo2drop_source: online
geo2drop_ipdeny_url: "https://www.ipdeny.com"
geo2drop_github_archive_url: "https://github.com/besmirzanaj/geo2drop/raw/data/all-zones.tar.gz"
geo2drop_archive_fallback_github: true
geo2drop_download_timeout: 30
geo2drop_config_dir: /etc/geo2drop
geo2drop_state_dir: /var/lib/geo2drop
geo2drop_zones_dir: "{{ geo2drop_state_dir }}/zones"
geo2drop_archive_dir: "{{ geo2drop_state_dir }}/archive"
geo2drop_bin_path: /usr/local/sbin/geo2drop-refresh
geo2drop_config_path: "{{ geo2drop_config_dir }}/geo2drop.conf"
geo2drop_force_recreate: false
geo2drop_apply_now: true
geo2drop_schedule_enabled: true
geo2drop_schedule_calendar: weekly
geo2drop_schedule_randomized_delay: 1h
geo2drop_schedule_persistent: true
```

Removed: `geo2drop_ipset_name`, `geo2drop_ipset_type`, `geo2drop_ipset_family`,
`geo2drop_ipset_maxelem`, `geo2drop_ipset_hashsize`, `geo2drop_firewalld_zone`,
`geo2drop_manage_firewalld`.

### `tasks/install.yml`

```yaml
- name: Install required packages
  ansible.builtin.package:
    name: "{{ geo2drop_packages }}"
    state: present

- name: Ensure nftables is enabled and running
  ansible.builtin.systemd:
    name: nftables
    state: started
    enabled: true
  when: geo2drop_manage_nftables | bool
```

### `tasks/main.yml`

Platform assertion already updated for Debian + EL. No further changes.

### `handlers/main.yml`

Replace firewalld-specific handlers:

```yaml
- name: Reload nftables
  ansible.builtin.command:
    cmd: nft --file /etc/nftables.conf
  changed_when: true
```

### `tasks/schedule.yml` and `tasks/apply.yml`

No changes — systemd timer and script execution are backend-agnostic.

### `templates/geo2drop-refresh.sh.j2`

Rewritten to use `nft` commands instead of `firewall-cmd` and `ipset`.
The idempotency logic (sha256, CHANGED/UNCHANGED) stays identical.

### `templates/geo2drop.conf.j2`

Variables renamed to match nftables naming (e.g. `SET_NAME` instead of
`IPSET_NAME`, `WHITELIST` added).

### `meta/main.yml`

Already updated for Debian. Add `nftables` to galaxy_tags.

## Repository Separation

```
besmirzanaj/geo2drop          ansible-role-geo2drop
├── README.md                 ├── README.md
├── geo2drop (standalone)     ├── tasks/
├── geo2drop.conf.example     ├── templates/
└── LICENSE                   │   ├── geo2drop-refresh.sh.j2 (based on fork)
                              │   └── geo2drop.conf.j2
                              ├── defaults/
                              ├── vars/
                              └── ...
```

The fork is a standalone project. The Ansible role ships its own copy of the
script as a template. The two can evolve independently — the role can be updated
to match the fork's latest version at any time, and the fork doesn't need to
know about Ansible.

## Verification

- `ansible-lint` passes on the role
- Molecule tests on EL8, EL9, Debian 12 (Docker)
- Manual: `nft list table inet geo2drop` to verify set contents and drop rules
- Manual: `geo2drop-refresh --dry-run` to verify entries without modifying state
- Manual: `geo2drop-refresh --force` to verify rebuild