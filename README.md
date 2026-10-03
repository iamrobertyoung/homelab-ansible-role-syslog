# homelab-ansible-role-syslog

[![Lint](https://github.com/iamrobertyoung/homelab-ansible-role-syslog/actions/workflows/lint.yml/badge.svg)](https://github.com/iamrobertyoung/homelab-ansible-role-syslog/actions/workflows/lint.yml)
[![Release](https://github.com/iamrobertyoung/homelab-ansible-role-syslog/actions/workflows/release.yml/badge.svg)](https://github.com/iamrobertyoung/homelab-ansible-role-syslog/actions/workflows/release.yml)
[![SLSA 3](https://slsa.dev/images/gh-badge-level3.svg)](https://slsa.dev)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/iamrobertyoung/homelab-ansible-role-syslog/badge)](https://scorecard.dev/viewer/?uri=github.com/iamrobertyoung/homelab-ansible-role-syslog)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Ansible role for configuring rsyslog to forward logs to one or more remote servers over TLS.

## Requirements

- Ansible >= 2.15
- Target: Ubuntu (focal, jammy, noble) or Debian (bullseye, bookworm)
- TLS certificates must be present on target hosts

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `syslog_destinations` | the in-cluster Loki receiver, see below | List of places to forward to. Each entry takes `host`, `port`, and optionally `framing`. |
| `syslog_fqdn` | **required** | Fully qualified domain name of the host (used for TLS certificate paths) |

```yaml
syslog_destinations:
  - host: syslog.k8s.local.iamrobertyoung.co.uk
    port: 6514
    framing: octet-counted
```

### Upgrading from v1.x

**The default destination changed from Graylog to Loki in v2.0.0**, so bumping
this role's version *is* the cutover for a host. That is deliberate: it makes
migrating a repo a one-line change rather than two rollouts. Pin `v1.2.0` if a
host is not ready to move.

`syslog_graylog_url` and `syslog_graylog_port` are gone. Nothing set them — the
twelve consuming repos all took the default — so removing them silently changes
nothing rather than silently ignoring an override. To keep forwarding to
Graylog, name it in `syslog_destinations` explicitly.

### Multiple destinations

A list, because a host forwards to two places while a destination is being
replaced — you add the new one, confirm logs arrive, then remove the old.

```yaml
syslog_destinations:
  - host: logs.example.com
    port: 5140
  - host: syslog.example.com
    port: 6514
    framing: octet-counted
```

### `framing`

Omitted by default, which leaves rsyslog's own default: non-transparent
framing, where messages are delimited by newlines. Set it to `octet-counted`
for a receiver that asks for one — [RFC 6587](https://tools.ietf.org/html/rfc6587#section-3.4.1)
calls that the recommended method, and without it the receiver must wait for
the *next* message to know the current one has ended, so a quiet host's last
line sits in limbo until it says something else.

## Usage

### Install via requirements.yml

```yaml
- src: git@github.com:iamrobertyoung/homelab-ansible-role-syslog.git
  scm: git
  version: main
  name: syslog
```

```bash
ansible-galaxy install -r requirements.yml
```

### Example Playbook

```yaml
- hosts: servers
  roles:
    - role: syslog
      vars:
        syslog_fqdn: "{{ inventory_hostname }}"
        syslog_destinations:
          - host: logs.example.com
            port: 5140
```

## What Gets Installed

- rsyslog with GnuTLS and RELP support
- `/etc/rsyslog.d/01-syslog-forward.conf`, forwarding all logs to every
  configured destination over TLS in RFC 5424 format
- Removal of `/etc/rsyslog.d/01-graylog.conf`, this role's previous
  single-destination file. rsyslog reads every file in `/etc/rsyslog.d`, so
  leaving it beside its replacement would forward every message twice.

Only `imjournal` is loaded, because the distribution's own `rsyslog.conf` does
not. `imuxsock` is already loaded there, `mmjsonparse` is used by no rule, and
there is no `gtls` module at all — the GnuTLS netstream driver is
`lmnsd_gtls.so`, loaded on demand by `StreamDriver="gtls"`. This role asked
for all three for a long time, and each failure made `rsyslogd -N1` exit
non-zero, so a config check could not be used as a gate.

## TLS Requirements

The role expects the following certificate files to exist on the target host:
- CA certificate: `/etc/ssl/certs/ca-certificates.crt`
- Host certificate: `/etc/ssl/certs/<syslog_fqdn>.crt`
- Host private key: `/etc/ssl/private/<syslog_fqdn>.pem`

## Supply Chain Security

This project implements [SLSA](https://slsa.dev/) Level 3 provenance for release artifacts.

- Provenance attestations are submitted to [GitHub Attestations](https://github.com/iamrobertyoung/homelab-ansible-role-syslog/attestations)
- Release artifacts include `.intoto.jsonl` provenance files
- Security posture tracked via [OpenSSF Scorecard](https://scorecard.dev/viewer/?uri=github.com/iamrobertyoung/homelab-ansible-role-syslog)

### Verifying Release Provenance

```bash
# Using GitHub CLI (recommended)
gh attestation verify syslog-<VERSION>.tar.gz \
  --repo iamrobertyoung/homelab-ansible-role-syslog

# Or using slsa-verifier
VERSION="v1.2.0"  # Replace with desired version
slsa-verifier verify-artifact syslog-${VERSION}.tar.gz \
  --provenance-path syslog-${VERSION}.tar.gz.intoto.jsonl \
  --source-uri github.com/iamrobertyoung/homelab-ansible-role-syslog \
  --source-tag "${VERSION}"
```

## License

MIT
