# Demo runbook (25 min) and troubleshooting

## Timeline
| Min | Step |
|---|---|
| 0-3 | Tour: topology, the workflow, inventory rings, Git repo |
| 3-9 | Run `00 Audit`; show ranked denials; explain label/port/boolean/module classes |
| 9-15 | `10 Remediate CHECK` with diff; show the plan output; approval node |
| 15-19 | `10 APPLY`, then `20 Enforce` on canary; show health gates |
| 19-25 | Injected fault: diagnose, fix, re-run |

## Diagnosis flow (never setenforce 0)
```bash
ausearch -m avc,user_avc -ts recent -i
sealert -a /var/log/audit/audit.log          # dnf install setroubleshoot-server (lab host)
ls -Z <path>; ps -eZ | grep <proc>
matchpathcon <path>
semanage fcontext -l | grep <path>
semanage port -l | grep <port>
getsebool -a | grep <app>
audit2allow -w -a                            # explain only
```

## Fault -> diagnosis -> fix
| Injected (`-e fault=`) | Evidence | Fix (survey override, then commit to Git) |
|---|---|---|
| `log_label` | `httpd_t` write denied on `var_t` in /srv/webapp/logs | `selinux_fcontexts_extra: [{path: /srv/webapp/logs, type: httpd_log_t}]` |
| `port` | `name_bind` denied for tcp/8480 | `selinux_ports_extra: [{port: 8480, type: http_port_t}]` |
| `boolean` | `name_connect` to DB port denied | `selinux_booleans_extra: [{name: httpd_can_network_connect_db, state: true}]` |
| `socket` | socket in /run/webapp labeled `var_run_t` | Re-run remediate (fcontext + tmpfiles + restorecon) |

Narrate the choice: the access is legitimate and narrow, so a label beats a boolean, and a boolean beats a module.

## Defense one-liners
- Quarterly-only denials: observe a full cycle; bridge with `semanage permissive -a <domain>` for that domain only.
- Custom modules: .te in Git, built in CI, reviewed by security, `.pp` installed with checksum; narrow to a dedicated domain over time.
- Disabled hosts: permissive in config, remove `selinux=0`, `.autorelabel`, reboot with console access, review AVCs, then enforce.
- Drift: root can always change it, so the control is detect and reverse in minutes (auditd MAC_STATUS, EDA, audited job), plus nightly check-mode reports.
- Workflows vs EDA: workflows orchestrate planned change; EDA reacts to events and calls the same audited templates.
