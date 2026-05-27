# ansible-role-geo2drop

Block traffic from selected countries on **Enterprise Linux 8/9/10** hosts
(RHEL, Rocky, AlmaLinux) using **firewalld + ipset**, sourcing IP ranges from
[ipdeny.com](https://www.ipdeny.com) with an optional GitHub-mirror fallback.

This is a native Ansible port of [m0zgen/geo2drop](https://github.com/m0zgen/geo2drop).
The role installs dependencies, renders a configuration file, deploys a small
refresh script, and schedules a systemd timer so the blocklist is rebuilt
periodically without re-running Ansible.

## What it does

1. Installs `firewalld`, `ipset`, `curl`, `tar`, `gzip`.
2. Enables and starts `firewalld`.
3. Renders `/etc/geo2drop/geo2drop.conf`.
4. Deploys `/usr/local/sbin/geo2drop-refresh` (the worker script).
5. Installs `geo2drop.service` + `geo2drop.timer` (weekly by default).
6. Runs the refresh script once to apply initial state:
   - Downloads country zone files (online / archive / local).
   - Computes a sha256 over the merged entry set; if it differs from the
     last-applied hash (or the ipset is missing), the ipset is rebuilt.
   - Attaches the ipset to firewalld's `drop` zone.
   - Reloads firewalld.

The refresh script is **idempotent**: subsequent runs print `UNCHANGED` and
exit without touching firewalld unless the upstream zone data has actually
changed.

## Requirements

- Target host: RHEL/Rocky/AlmaLinux 8, 9, or 10.
- Ansible 2.14+ on the controller.
- Outbound HTTPS to `www.ipdeny.com` (or `github.com` if archive fallback
  is enabled).

## Role Variables

All variables are defined in `defaults/main.yml`. The most useful ones:

| Variable | Default | Description |
|----------|---------|-------------|
| `geo2drop_countries` | `[br, id, cl]` | ISO 3166-1 alpha-2 codes to block. |
| `geo2drop_ipset_name` | `blcountries` | Name of the firewalld ipset. |
| `geo2drop_ipset_family` | `inet` | `inet` for IPv4, `inet6` for IPv6. |
| `geo2drop_ipset_maxelem` | `131072` | Max entries in the ipset. |
| `geo2drop_ipset_hashsize` | `4096` | Initial hash bucket count. |
| `geo2drop_firewalld_zone` | `drop` | Firewalld zone the ipset is bound to. |
| `geo2drop_source` | `online` | `online` \| `archive` \| `local`. |
| `geo2drop_archive_fallback_github` | `true` | If `archive` mode and ipdeny is down, use the GitHub mirror. |
| `geo2drop_force_recreate` | `false` | Force rebuild even when entries are unchanged. |
| `geo2drop_apply_now` | `true` | Run the refresh script once during the play. |
| `geo2drop_schedule_enabled` | `true` | Install the systemd timer. |
| `geo2drop_schedule_calendar` | `weekly` | systemd `OnCalendar=` expression. |
| `geo2drop_schedule_randomized_delay` | `1h` | Spread runs across a fleet. |

Source modes:

- **online** — fetch each `<country>.zone` from ipdeny.com on every refresh.
  Lowest disk usage, more requests.
- **archive** — fetch `all-zones.tar.gz` once and extract. Falls back to the
  GitHub mirror if ipdeny is unreachable (recommended for restricted regions).
- **local** — operator pre-places `<country>.zone` files in
  `/var/lib/geo2drop/zones/`; the role never downloads.

## Installation

Install via `ansible-galaxy` with a `requirements.yml`:

```yaml
---
roles:
  - src: https://github.com/besmirzanaj/ansible-role-geo2drop
    name: besmirzanaj.geo2drop
    version: main
```

```
ansible-galaxy role install -r requirements.yml
```

## Example Playbook

```yaml
- hosts: edge_hosts
  become: true
  roles:
    - role: besmirzanaj.geo2drop
      vars:
        geo2drop_countries: [br, cn, in, id, ru]
        geo2drop_source: archive
        geo2drop_schedule_calendar: "Mon *-*-* 03:00:00"
```

Run once without scheduling:

```yaml
- hosts: edge_hosts
  become: true
  roles:
    - role: besmirzanaj.geo2drop
      vars:
        geo2drop_schedule_enabled: false
```

Force a rebuild on the next run:

```yaml
- hosts: edge_hosts
  become: true
  roles:
    - role: besmirzanaj.geo2drop
      vars:
        geo2drop_force_recreate: true
```

## Operational tips

- Inspect current state on a target:
  ```
  sudo firewall-cmd --permanent --get-ipsets
  sudo firewall-cmd --permanent --ipset=blcountries --get-entries | head
  sudo firewall-cmd --permanent --zone=drop --list-sources
  ```
- Trigger an out-of-cycle refresh:
  ```
  sudo systemctl start geo2drop.service
  sudo journalctl -u geo2drop.service -n 50
  ```
- Force a full rebuild without re-running Ansible:
  ```
  sudo /usr/local/sbin/geo2drop-refresh --force
  ```
- Remove everything: set `geo2drop_schedule_enabled: false`, run, then
  manually delete the ipset:
  ```
  sudo firewall-cmd --permanent --zone=drop --remove-source=ipset:blcountries
  sudo firewall-cmd --permanent --delete-ipset=blcountries
  sudo firewall-cmd --reload
  ```

## Notes on idempotency

`firewall-cmd --add-entries-from-file` is not idempotent on its own. To avoid
churn (and unnecessary firewalld reloads), the refresh script:

1. Builds a sorted, de-duplicated entry file from the country zones.
2. Hashes it (sha256) and compares against `/var/lib/geo2drop/applied.sha256`.
3. Only rebuilds the ipset when the hash differs or the ipset is missing.
4. Emits `CHANGED` / `UNCHANGED` so Ansible's `changed_when` is accurate.

## License

MIT.

## Author

Native Ansible port maintained by Besmir Zanaj. Upstream:
[m0zgen/geo2drop](https://github.com/m0zgen/geo2drop).
