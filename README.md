# nuno.owasp — OWASP Top 10:2025 on Ansible Automation Platform

System and platform controls for the OWASP categories Ansible can actually enforce. This collection does **not** fix application bugs such as injection, IDOR, or insecure design.

| Playbook | OWASP | What it does |
| --- | --- | --- |
| `playbooks/baseline_harden.yml` | A01, A02, A07 | firewalld, SELinux enforcing, SSH drop-in, account file modes, httpd `-Indexes` |
| `playbooks/inventory_packages.yml` | A03, A08 | package inventory, security-update list, `gpgcheck=1` |
| `playbooks/patch_packages.yml` | A03 | security-only DNF updates |
| `playbooks/crypto_tls.yml` | A04 | `update-crypto-policies --set` |
| `playbooks/logging_alert.yml` | A09 | auditd, rsyslog, identity/auth audit watches |
| `playbooks/drift_scan.yml` | A02, A08 | OpenSCAP profile eval; job fails on failed rules |
| `playbooks/assess.yml` | A03 + A02/A08 | inventory then drift scan |
| `playbooks/remediate.yml` | A02/A04/A07/A09 | baseline, crypto, logging. Does not patch |
| `playbooks/site.yml` | assess | same as `assess.yml` |
| `playbooks/install.yml` | AAP | job templates and workflows on Controller |

A05 Injection and A06 Insecure Design are out of scope. A10 is only covered indirectly (sshd validate, fail-closed OpenSCAP).

## Lab inventory

Edit `inventories/lab/hosts.yml`. `rhel9-lab` is an RFC 5737 placeholder. Point it at a **recoverable RHEL 9+ host**. Do not run `baseline_harden` against a workstation you cannot get back into: it disables SSH password and root login.

```sh
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/assess.yml -i inventories/lab/hosts.yml
ansible-playbook playbooks/remediate.yml -i inventories/lab/hosts.yml
```

Reports land on the target under `/var/tmp/owasp-*.md` and are fetched to `artifacts/<hostname>/`.

## Install onto Controller

Requires Ansible Automation Platform 2.5+ and an execution environment that includes `ansible.posix` (for example `ee-supported-rhel9`).

1. Create a **machine** credential and a **Red Hat Ansible Automation Platform** credential. Install does not create secrets.
2. Create a Controller project named `aap_owasp` that tracks this git repository.
3. Copy `vars/aap.yml.example` to `vars/aap.yml` and set organization, inventory, project, credential, and execution-environment names.
4. Run:

```sh
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_OAUTH_TOKEN=...
export CONTROLLER_VERIFY_SSL=true

ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/install.yml -e @vars/aap.yml
```

`vars/aap.yml` is gitignored.

Objects created:

| Object | Name |
| --- | --- |
| Label | `owasp-top10` |
| Job template | `OWASP A02 - Baseline harden` |
| Job template | `OWASP A03 - Inventory packages` |
| Job template | `OWASP A03 - Patch security updates` |
| Job template | `OWASP A04 - Crypto and TLS` |
| Job template | `OWASP A09 - Logging and audit` |
| Job template | `OWASP A02 A08 - Drift scan` |
| Workflow | `OWASP - Assess` |
| Workflow | `OWASP - Remediate` |

Patch stays a standalone job. It is not in the remediate workflow.

## Event-Driven Ansible

`extensions/eda/rulebooks/owasp_auth_alerts.yml` watches journald for failed SSH passwords (A07/A09). Wire the debug action to a Controller job template when you attach an AAP decision environment.

## Execution environment

```sh
ansible-builder build -f execution-environment.yml -t aap-owasp:1.0.0
```
