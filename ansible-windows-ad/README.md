# Ansible Windows AD (Server 2022)

Deploy Active Directory Domain Services from Ubuntu to a Windows Server 2022 VM over WinRM.

## What it does

- Installs AD DS + DNS role
- Promotes the host as first Domain Controller (new forest)
- Sets DNS forwarders
- Creates OUs required by documentation: `Angajati`, `GrupuriSecuritate`, `ServiceAccounts`
- Creates AD security groups: `SG_ClusterAdmins`, `SG_PlatformEngineers`, `SG_JuniorDevs`, `SG_NonTechnical`
- Creates test personas: `alice`, `sabin`, `bob`, `eve`
- Creates bind account: `svc_keycloak_bind`

## Prerequisites

- Ubuntu control node with Ansible
- Reachability to Windows VM on WinRM (`5986` preferred)
- Local Administrator credentials on Windows VM
- Python package `pywinrm` on Ubuntu

## 1) Install dependencies

```bash
cd ansible-playbooks/ansible-windows-ad
ansible-galaxy collection install -r requirements.yml
python3 -m pip install --user pywinrm
```

## 2) Edit inventory and vars

- `inventories/prod/hosts.yml`
- `group_vars/windows_ad.yml`

Replace all `CHANGEME_*` values.

Default values are aligned with the documentation under:

- `infrastructure-deploy/keycloak-oauth2proxy-ad-DOC/ldap.md`
- Base DN: `DC=licenta,DC=ro`

## 3) (Optional) Bootstrap WinRM on target

Run only if WinRM is not already configured:

```bash
ansible-playbook playbooks/01-bootstrap-winrm.yml
```

## 4) Install AD DS and bootstrap LDAP objects

```bash
ansible-playbook playbooks/site.yml
```

## Notes

- For production, store passwords with `ansible-vault`.
- Use a proper TLS cert on WinRM HTTPS listener.
- If the server is already a DC, rerunning remains mostly idempotent.
