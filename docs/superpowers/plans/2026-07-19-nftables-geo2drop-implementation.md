# nftables geo2drop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace firewalld+ipset backend with native nftables sets in both the standalone `besmirzanaj/geo2drop` fork and the `ansible-role-geo2drop` Ansible role.

**Architecture:** The fork is a standalone bash script managing nftables table `inet geo2drop` with two sets (`blcountries`, `blcountries6`) and two chains (`geo2drop_input`, `geo2drop_forward`). The Ansible role templatizes the same script, adds package installation, config rendering, and systemd timer scheduling.

**Tech Stack:** bash, nftables, Ansible, systemd

## Global Constraints

- Works on EL8/9/10 and Debian 11/12/13
- No firewalld dependency
- nftables table `inet geo2drop` with priority `-10`
- `policy accept` on all chains — only matched packets are dropped
- Idempotent: sha256 of sorted entries compared against `applied.sha256`
- Whitelist support: CIDRs excluded from the set before population
- `auto-merge` on set flags
- `counter` on drop rules for visibility
- `nft -f` with temp file for batch element insertion (avoids CLI limits)
- Systemd timer for periodic refresh (same as current)
- Config file at `/etc/geo2drop/geo2drop.conf` (same as current)

---

## Phase 1: Fork `besmirzanaj/geo2drop`

### Task 1: Create standalone bash script

**Files:**
- Create: `geo2drop` (standalone script, executable)

**Interfaces:**
- Consumes: config file at `/etc/geo2drop/geo2drop.conf` (or `--config PATH`)
- Produces: nftables table `inet geo2drop` with sets and drop rules

- [ ] **Step 1: Write the standalone script `geo2drop`**

```bash
#!/bin/bash
set -euo pipefail

CONFIG="${GEO2DROP_CONF:-/etc/geo2drop/geo2drop.conf}"
FORCE=0
DRY_RUN=0

usage() {
    cat <<EOF
Usage: $0 [--config PATH] [--force] [--dry-run]

  --config PATH   Override config path (default: ${CONFIG})
  --force         Rebuild the nftables set even if entries are unchanged
  --dry-run       Print what would be done without modifying nftables
  -h, --help      Show this help
EOF
}

while [[ $# -gt 0 ]]; do
    case "$1" in
        --config) CONFIG="$2"; shift ;;
        --force) FORCE=1 ;;
        --dry-run) DRY_RUN=1 ;;
        -h|--help) usage; exit 0 ;;
        *) echo "Unknown argument: $1" >&2; usage >&2; exit 1 ;;
    esac
    shift
done

if [[ ! -r "${CONFIG}" ]]; then
    echo "ERROR: cannot read config ${CONFIG}" >&2
    exit 1
fi
# shellcheck disable=SC1090
. "${CONFIG}"

: "${COUNTRIES:?COUNTRIES must be set in ${CONFIG}}"
: "${SET_NAME:=blcountries}"
: "${SET_NAME6:=${SET_NAME}6}"
: "${TABLE_FAMILY:=inet}"
: "${TABLE_NAME:=geo2drop}"
: "${SOURCE_MODE:=online}"
: "${IPDENY_URL:=https://www.ipdeny.com}"
: "${GITHUB_ARCHIVE_URL:=https://github.com/besmirzanaj/geo2drop/raw/data/all-zones.tar.gz}"
: "${ARCHIVE_FALLBACK_GITHUB:=true}"
: "${DOWNLOAD_TIMEOUT:=30}"
: "${STATE_DIR:=/var/lib/geo2drop}"
: "${ZONES_DIR:=${STATE_DIR}/zones}"
: "${ARCHIVE_DIR:=${STATE_DIR}/archive}"
: "${WHITELIST:=}"

STATE_FILE="${STATE_DIR}/applied.sha256"
DESIRED_FILE="$(mktemp -t geo2drop.XXXXXX)"
DESIRED_FILE6="$(mktemp -t geo2drop6.XXXXXX)"
NFT_FILE="$(mktemp -t geo2drop-nft.XXXXXX)"
trap 'rm -f "${DESIRED_FILE}" "${DESIRED_FILE6}" "${NFT_FILE}"' EXIT

log() { echo "[geo2drop] $*"; }
die() { echo "[geo2drop] ERROR: $*" >&2; exit 1; }

require_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "missing required command: $1"
}

require_cmd curl
require_cmd tar
require_cmd nft
require_cmd sha256sum
require_cmd sort
require_cmd grep

mkdir -p "${STATE_DIR}" "${ZONES_DIR}" "${ARCHIVE_DIR}"

# --- Zone fetching (same as current, unchanged) ---

is_url_available() {
    curl -sSfI --max-time "${DOWNLOAD_TIMEOUT}" "$1" >/dev/null 2>&1
}

fetch_online() {
    is_url_available "${IPDENY_URL}" \
        || die "ipdeny.com is not reachable (source=online)"
    local country url
    for country in ${COUNTRIES}; do
        url="${IPDENY_URL}/ipblocks/data/countries/${country}.zone"
        log "downloading ${country} from ${url}"
        if ! curl -sSf --max-time "${DOWNLOAD_TIMEOUT}" "${url}" -o "${ZONES_DIR}/${country}.zone.new"; then
            die "failed to download ${url}"
        fi
        mv -f "${ZONES_DIR}/${country}.zone.new" "${ZONES_DIR}/${country}.zone"
    done
    if [[ "${TABLE_FAMILY}" != "ip" ]]; then
        for country in ${COUNTRIES}; do
            url="${IPDENY_URL}/ipv6/ipblocks/data/countries/${country}.zone"
            log "downloading ${country} IPv6 from ${url}"
            if ! curl -sSf --max-time "${DOWNLOAD_TIMEOUT}" "${url}" -o "${ZONES_DIR}/${country}.zone6.new"; then
                die "failed to download IPv6 ${url}"
            fi
            mv -f "${ZONES_DIR}/${country}.zone6.new" "${ZONES_DIR}/${country}.zone6"
        done
    fi
}

fetch_archive() {
    local archive="${ARCHIVE_DIR}/all-zones.tar.gz"
    local src=""
    if is_url_available "${IPDENY_URL}/ipblocks/data/countries/all-zones.tar.gz"; then
        src="${IPDENY_URL}/ipblocks/data/countries/all-zones.tar.gz"
    elif [[ "${ARCHIVE_FALLBACK_GITHUB}" == "true" ]] \
        && is_url_available "${GITHUB_ARCHIVE_URL}"; then
        log "ipdeny.com unreachable; falling back to GitHub mirror"
        src="${GITHUB_ARCHIVE_URL}"
    else
        die "no reachable archive source"
    fi
    log "downloading archive from ${src}"
    if ! curl -sSfL --max-time "$((DOWNLOAD_TIMEOUT * 4))" "${src}" -o "${archive}.new"; then
        die "failed to download ${src}"
    fi
    mv -f "${archive}.new" "${archive}"
    log "extracting archive to ${ZONES_DIR}"
    tar -xzf "${archive}" -C "${ZONES_DIR}"
}

fetch_zones() {
    case "${SOURCE_MODE}" in
        online)  fetch_online ;;
        archive) fetch_archive ;;
        local)   log "source=local; using existing files in ${ZONES_DIR}" ;;
        *) die "invalid SOURCE_MODE=${SOURCE_MODE}" ;;
    esac
}

# --- Entry building ---

build_desired_entries() {
    local country missing=0
    : > "${DESIRED_FILE}"
    for country in ${COUNTRIES}; do
        local f="${ZONES_DIR}/${country}.zone"
        if [[ ! -s "${f}" ]]; then
            log "WARN: zone file missing or empty for '${country}' (${f})"
            missing=$((missing + 1))
            continue
        fi
        grep -Ev '^[[:space:]]*(#|$)' "${f}" | sed "s/$/,/" >> "${DESIRED_FILE}" || true
    done
    if [[ ! -s "${DESIRED_FILE}" ]]; then
        die "no entries collected (${missing} country file(s) missing)"
    fi
    sort -u -o "${DESIRED_FILE}" "${DESIRED_FILE}"
}

build_desired_entries6() {
    local country missing=0 f
    : > "${DESIRED_FILE6}"
    for country in ${COUNTRIES}; do
        f="${ZONES_DIR}/${country}.zone6"
        if [[ ! -s "${f}" ]]; then
            log "WARN: IPv6 zone file missing for '${country}' (${f})"
            missing=$((missing + 1))
            continue
        fi
        grep -Ev '^[[:space:]]*(#|$)' "${f}" | sed "s/$/,/" >> "${DESIRED_FILE6}" || true
    done
    if [[ -s "${DESIRED_FILE6}" ]]; then
        sort -u -o "${DESIRED_FILE6}" "${DESIRED_FILE6}"
    fi
}

filter_whitelist() {
    local cidr
    for cidr in ${WHITELIST}; do
        sed -i "\|^${cidr},$|d" "${DESIRED_FILE}" 2>/dev/null || true
        sed -i "\|^${cidr},$|d" "${DESIRED_FILE6}" 2>/dev/null || true
    done
}

# --- Nftables management ---

table_exists() {
    nft list table "${TABLE_FAMILY}" "${TABLE_NAME}" >/dev/null 2>&1
}

set_exists() {
    nft list set "${TABLE_FAMILY}" "${TABLE_NAME}" "$1" >/dev/null 2>&1
}

create_table_structure() {
    log "ensuring table ${TABLE_FAMILY} ${TABLE_NAME} exists"
    nft add table "${TABLE_FAMILY}" "${TABLE_NAME}" 2>/dev/null || true

    if ! set_exists "${SET_NAME}"; then
        log "creating set ${SET_NAME}"
        nft add set "${TABLE_FAMILY}" "${TABLE_NAME}" "${SET_NAME}" \
            '{ type ipv4_addr; flags interval; auto-merge; }'
    fi

    if [[ "${TABLE_FAMILY}" != "ip" ]] && ! set_exists "${SET_NAME6}"; then
        log "creating set ${SET_NAME6}"
        nft add set "${TABLE_FAMILY}" "${TABLE_NAME}" "${SET_NAME6}" \
            '{ type ipv6_addr; flags interval; auto-merge; }'
    fi

    if ! nft list chain "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_input >/dev/null 2>&1; then
        log "creating chain geo2drop_input"
        nft add chain "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_input \
            '{ type filter hook input priority -10; policy accept; }'
        nft add rule "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_input \
            "ip saddr @${SET_NAME} counter drop"
        if [[ "${TABLE_FAMILY}" != "ip" ]]; then
            nft add rule "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_input \
                "ip6 saddr @${SET_NAME6} counter drop"
        fi
    fi

    if ! nft list chain "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_forward >/dev/null 2>&1; then
        log "creating chain geo2drop_forward"
        nft add chain "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_forward \
            '{ type filter hook forward priority -10; policy accept; }'
        nft add rule "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_forward \
            "ip saddr @${SET_NAME} counter drop"
        if [[ "${TABLE_FAMILY}" != "ip" ]]; then
            nft add rule "${TABLE_FAMILY}" "${TABLE_NAME}" geo2drop_forward \
                "ip6 saddr @${SET_NAME6} counter drop"
        fi
    fi
}

populate_set() {
    local set_name="$1"
    local data_file="$2"
    if [[ ! -s "${data_file}" ]]; then
        log "no entries to load into ${set_name}; skipping"
        return
    fi
    log "populating ${set_name} from $(wc -l < "${data_file}") entries"
    {
        echo "flush set ${TABLE_FAMILY} ${TABLE_NAME} ${set_name}"
        echo "add element ${TABLE_FAMILY} ${TABLE_NAME} ${set_name} {"
        cat "${data_file}"
        echo "}"
    } > "${NFT_FILE}"

    if [[ "${DRY_RUN}" -eq 1 ]]; then
        log "DRY RUN: would apply nft ruleset from ${NFT_FILE}"
        cat "${NFT_FILE}"
    else
        nft -f "${NFT_FILE}"
    fi
}

desired_hash() {
    # Hash the combined CIDR entries (v4 + v6), stripping trailing commas
    {
        sed 's/,$//' "${DESIRED_FILE}"
        [[ -s "${DESIRED_FILE6}" ]] && sed 's/,$//' "${DESIRED_FILE6}"
    } | sha256sum | awk '{print $1}'
}

# --- Main ---

main() {
    log "countries: ${COUNTRIES}"
    if [[ -n "${WHITELIST}" ]]; then
        log "whitelist: ${WHITELIST}"
    fi
    fetch_zones
    build_desired_entries
    if [[ "${TABLE_FAMILY}" != "ip" ]] && [[ "${SOURCE_MODE}" != "local" ]]; then
        build_desired_entries6
    fi
    filter_whitelist

    local new_hash old_hash="" need_apply=0
    new_hash="$(desired_hash)"
    [[ -f "${STATE_FILE}" ]] && old_hash="$(cat "${STATE_FILE}")"

    if [[ "${FORCE}" -eq 1 ]]; then
        log "force mode: rebuilding nftables set"
        need_apply=1
    elif ! table_exists || ! set_exists "${SET_NAME}"; then
        log "nftables table or set missing: building"
        need_apply=1
    elif [[ "${new_hash}" != "${old_hash}" ]]; then
        log "entry set changed (old=${old_hash:-<none>} new=${new_hash}): rebuilding"
        need_apply=1
    else
        log "entry set unchanged; verifying table structure only"
    fi

    if [[ "${need_apply}" -eq 1 ]]; then
        if [[ "${DRY_RUN}" -eq 1 ]]; then
            log "DRY RUN: would rebuild set with ${new_hash}"
        else
            create_table_structure
            populate_set "${SET_NAME}" "${DESIRED_FILE}"
            if [[ "${TABLE_FAMILY}" != "ip" ]]; then
                populate_set "${SET_NAME6}" "${DESIRED_FILE6}"
            fi
            echo "${new_hash}" > "${STATE_FILE}"
        fi
        echo "CHANGED"
    else
        create_table_structure
        echo "UNCHANGED"
    fi
}

main "$@"
```

- [ ] **Step 2: Make script executable**

```bash
chmod +x geo2drop
```

- [ ] **Step 3: Verify script parses correctly**

```bash
bash -n geo2drop
```

Expected: no output (syntax OK)

- [ ] **Step 4: Commit**

```bash
git add geo2drop
git commit -m "feat: standalone nftables-based geoip blocking script"
```

### Task 2: Create example config, README, LICENSE

**Files:**
- Create: `geo2drop.conf.example`
- Create: `README.md`
- Create: `LICENSE`

- [ ] **Step 1: Write `geo2drop.conf.example`**

```
COUNTRIES="br id cl"
WHITELIST=""
SET_NAME="blcountries"
TABLE_FAMILY="inet"
SOURCE_MODE="online"
IPDENY_URL="https://www.ipdeny.com"
GITHUB_ARCHIVE_URL="https://github.com/besmirzanaj/geo2drop/raw/data/all-zones.tar.gz"
ARCHIVE_FALLBACK_GITHUB="true"
DOWNLOAD_TIMEOUT="30"
STATE_DIR="/var/lib/geo2drop"
ZONES_DIR="${STATE_DIR}/zones"
ARCHIVE_DIR="${STATE_DIR}/archive"
```

- [ ] **Step 2: Write `README.md`**

```markdown
# geo2drop — nftables-based geo IP blocking

Block traffic from selected countries using nftables sets. Fetches country
IP ranges from [ipdeny.com](https://www.ipdeny.com) and creates nftables
drop rules in the `inet geo2drop` table.

## Requirements

- nftables (kernel + userspace)
- curl, tar, gzip

## Installation

```
sudo cp geo2drop /usr/local/bin/
sudo cp geo2drop.conf.example /etc/geo2drop/geo2drop.conf
# Edit /etc/geo2drop/geo2drop.conf with your countries
sudo geo2drop
```

## Usage

```
geo2drop [--config PATH] [--force] [--dry-run]
```

- `--config PATH` — override config path (default `/etc/geo2drop/geo2drop.conf`)
- `--force` — rebuild the set even if entries are unchanged
- `--dry-run` — print what would be done without modifying nftables

## Config

See `geo2drop.conf.example` for all variables.

## License

MIT
```

- [ ] **Step 3: Write `LICENSE`** (MIT, same as upstream)

- [ ] **Step 4: Commit**

```bash
git add geo2drop.conf.example README.md LICENSE
git commit -m "docs: add example config, README, license"
```

---

## Phase 2: Ansible Role Changes

### Task 3: Update `vars/main.yml` — package list

**Files:**
- Modify: `vars/main.yml`

- [ ] **Step 1: Replace package list**

```yaml
---
geo2drop_packages:
  - nftables
  - curl
  - tar
  - gzip
```

- [ ] **Step 2: Commit**

```bash
git add vars/main.yml
git commit -m "refactor: replace firewalld/ipset packages with nftables"
```

### Task 4: Update `defaults/main.yml` — nftables variables

**Files:**
- Modify: `defaults/main.yml`

- [ ] **Step 1: Rewrite defaults**

```yaml
---
geo2drop_countries:
  - br
  - id
  - cl

geo2drop_whitelist: []

geo2drop_set_name: blcountries
geo2drop_nft_family: inet

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

geo2drop_manage_nftables: true
```

- [ ] **Step 2: Commit**

```bash
git add defaults/main.yml
git commit -m "refactor: replace firewalld/ipset defaults with nftables variables"
```

### Task 5: Update `tasks/install.yml` — nftables service

**Files:**
- Modify: `tasks/install.yml`

- [ ] **Step 1: Replace firewalld with nftables**

```yaml
---
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

- [ ] **Step 2: Commit**

```bash
git add tasks/install.yml
git commit -m "refactor: install nftables instead of firewalld"
```

### Task 6: Rewrite `templates/geo2drop-refresh.sh.j2` — nftables logic

**Files:**
- Modify: `templates/geo2drop-refresh.sh.j2`

- [ ] **Step 1: Write the nftables-based refresh script template**

The template is identical to the standalone `geo2drop` script from Task 1, with only `{{ ansible_managed }}` added. The script sources config values from the rendered config file at runtime (same pattern as the current role). No Jinja2 variables are needed for config values — they come from the config file sourced at runtime.

The only difference from the standalone script:
- `{{ ansible_managed | comment(decoration='# ') }}` header
- Default config path points to `{{ geo2drop_config_path }}` (rendered to `/etc/geo2drop/geo2drop.conf`)

The script logic is the same as Task 1 (nftables commands, sha256 idempotency, whitelist filtering).

- [ ] **Step 2: Commit**

```bash
git add templates/geo2drop-refresh.sh.j2
git commit -m "feat: rewrite refresh script for nftables backend"
```

### Task 7: Update `templates/geo2drop.conf.j2` — rename variables

**Files:**
- Modify: `templates/geo2drop.conf.j2`

- [ ] **Step 1: Rewrite config template**

```
{{ ansible_managed | comment }}
# /etc/geo2drop/geo2drop.conf

COUNTRIES="{{ geo2drop_countries | join(' ') }}"
WHITELIST="{{ geo2drop_whitelist | join(' ') }}"

SET_NAME="{{ geo2drop_set_name }}"
TABLE_FAMILY="{{ geo2drop_nft_family }}"

SOURCE_MODE="{{ geo2drop_source }}"

IPDENY_URL="{{ geo2drop_ipdeny_url }}"
GITHUB_ARCHIVE_URL="{{ geo2drop_github_archive_url }}"
ARCHIVE_FALLBACK_GITHUB="{{ 'true' if geo2drop_archive_fallback_github else 'false' }}"
DOWNLOAD_TIMEOUT="{{ geo2drop_download_timeout }}"

STATE_DIR="{{ geo2drop_state_dir }}"
ZONES_DIR="{{ geo2drop_zones_dir }}"
ARCHIVE_DIR="{{ geo2drop_archive_dir }}"
```

- [ ] **Step 2: Commit**

```bash
git add templates/geo2drop.conf.j2
git commit -m "refactor: rename ipset vars to nftables vars in config template"
```

### Task 8: Update `handlers/main.yml` — remove firewalld handlers

**Files:**
- Modify: `handlers/main.yml`

- [ ] **Step 1: Replace firewalld handlers with nftables handler**

```yaml
---
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: Restart geo2drop timer
  ansible.builtin.systemd:
    name: geo2drop.timer
    state: restarted
    enabled: true
```

Removed: `Reload firewalld`, `Restart firewalld` (nft changes are immediate, no reload needed).

- [ ] **Step 2: Commit**

```bash
git add handlers/main.yml
git commit -m "refactor: remove firewalld handlers, keep systemd handlers"
```

### Task 9: Update `meta/main.yml` — add nftables tag

**Files:**
- Modify: `meta/main.yml`

- [ ] **Step 1: Add nftables to galaxy_tags**

```yaml
  galaxy_tags:
    - firewall
    - nftables
    - ipset
    - security
    - geoip
    - rhel
    - rocky
    - almalinux
    - debian
```

- [ ] **Step 2: Commit**

```bash
git add meta/main.yml
git commit -m "chore: add nftables tag to galaxy metadata"
```

### Task 10: Update `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update documentation for nftables**

Update the description, requirements, variable table, and usage examples to reflect nftables instead of firewalld+ipset.

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README for nftables backend"
```

### Task 11: Update moleculer config for Debian testing

**Files:**
- Modify: `molecule/default/molecule.yml`

- [ ] **Step 1: Add Debian 12 platform and update converge.yml**

Update `molecule/default/molecule.yml`:
```yaml
  - name: geo2drop-debian12
    image: debian:12
    pre_build_image: true
    command: /usr/sbin/init
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
```

Update `molecule/default/converge.yml` to use the new variable name:
```yaml
    geo2drop_manage_nftables: false
    geo2drop_apply_now: false
    geo2drop_schedule_enabled: false
```

- [ ] **Step 2: Commit**

```bash
git add molecule/default/molecule.yml
git commit -m "ci: add Debian 12 test platform"
```

### Task 12: Run ansible-lint and verify

- [ ] **Step 1: Run ansible-lint**

```bash
cd /Users/bzanaj/git/ansible-role-geo2drop && ansible-lint
```

Expected: Pass (0 failures, 0 warnings)

- [ ] **Step 2: Fix any issues found**

- [ ] **Step 3: Final commit if fixes were needed**

```bash
git commit -m "fix: address lint issues"
```