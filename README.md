# ACME Financial: SELinux Enforcement with Ansible Automation Platform

Evidence-driven, ring-based move from Permissive to Enforcing on 1,200 RHEL 9 nodes, with continuous drift control.
All code is a tested-by-reading starting point: run it in your RHDP staging environment and fix versions and names before the panel.

## Flow

| Step | Playbook | Controller template | Purpose |
|---|---|---|---|
| 0 | `00_audit.yml` | SELinux 00 Audit | Rank denials fleet-wide (source, target, class, permission) |
| 1 | `10_remediate.yml` | 10 CHECK (check + diff) then 10 APPLY | sefcontext, seport, seboolean, tmpfiles, relabel |
| 2 | `50_custom_modules.yml` | 50 Custom Modules | Only when no label/port/boolean fits |
| 3 | `20_enforce.yml` | 20 Enforce | Gate, drain, flip, health + denial checks, auto safety net |
| 4 | `40_compliance.yml` | 40 REPORT / 40 Drift REMEDIATE | Config, kernel args, auditd rules, optional OpenSCAP |
| - | `30_disabled_to_enforcing.yml` | 30 Disabled Migration | Disabled hosts: permissive, relabel, reboot (serial 1) |
| - | `90_inject_fault.yml` | 90 Inject Fault | Demo helper for live troubleshooting |

`controller/configure.yml` builds the project, templates, the rollout workflow (with a CISO approval node) and a nightly report.
`rulebooks/selinux_drift.yml` is the EDA rulebook that reverses `setenforce 0` within minutes.

## Quick start

```bash
ansible-galaxy collection install -r collections/requirements.yml
# edit inventories/rhdp/hosts.yml and group_vars/{web,db}.yml for your hosts, paths and ports
ansible-playbook playbooks/00_audit.yml
ansible-playbook playbooks/10_remediate.yml --check --diff
ansible-playbook playbooks/10_remediate.yml
ansible-playbook playbooks/20_enforce.yml -e ring=ring_canary
```

Build the EE from `execution-environment/` (ansible-builder) because community.general is not in the supported EE.
Alternative: the supported `redhat.rhel_system_roles.selinux` role covers fcontexts, ports, booleans and modules.

## Design rules

- Label first, boolean second, custom module last. Never apply `audit2allow` output unreviewed.
- Variables per tier live in `group_vars` (Git is the source of truth). `*_extra` survey variables allow a live fix, then commit it.
- Enforcing is a runtime flip (no reboot). Only disabled hosts reboot, and only through playbook 30.
- Dry run caveat: in check mode new mappings are not applied, so the relabel preview cannot see them. The plan task diffs desired vs `semanage ... -l -C` instead.
- The safety-net rollback (rescue sets Permissive) is for the rollout only. During live troubleshooting fix the cause and never go permissive.

## Known things to verify in your environment

- Module/parameter names against your collection versions (`ansible.controller` workflow and approval node fields, `community.general.counter` filter).
- Container hosts need `python3-libselinux`; `/usr/sbin` is a symlink to `/usr/bin` on RHEL 9, so confirm the auditd path watches load (`auditctl -l`).
- EDA source: the Kafka path needs the rsyslog JSON template noted in the rulebook; the webhook fallback is simpler in a lab.
