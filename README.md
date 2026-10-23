# Octopus Tentacle Ansible role (non-root, RHEL 7 to 10 and Windows Server)

This repo contains an Ansible role, `octopus_tentacle`, that installs an Octopus Deploy Listening Tentacle on RHEL 7, 8, 9 or 10 so that it runs as an ordinary user instead of root. It follows the [rootless instance](https://octopus.com/docs/infrastructure/deployment-targets/tentacle/linux#rootless-instance-creation) approach from the Octopus docs, and you can optionally register the machine with your Octopus Server as a deployment target or a worker.

The same role also installs Tentacle on Windows Server, running the Windows service as a local non-admin account. See [Windows hosts](#windows-hosts).

One role covers all four RHEL versions. It reads the host's major version and picks the Tentacle build to match: RHEL 7 gets 8.1.2007, the newest Tentacle known to run there, and RHEL 8, 9 and 10 get whatever you set (the default is `latest`).

On Linux it only uses modules that ship with `ansible-core`, so Linux-only users don't need any collections. Windows hosts need the `ansible.windows` collection (see [Windows hosts](#windows-hosts)). The same code runs from a laptop with the free `ansible-core` package or from Red Hat Ansible Automation Platform (AAP).

> [!WARNING]
> This project is experimental and is not an official Octopus Deploy product. Octopus Support can't help with problems you hit while using it. If something goes wrong, please raise an [issue](https://github.com/OctopusDeployLabs/ansible-role-octopus-tentacle/issues) instead (see [Support](#support)).

## What it does on each host

Ansible connects as an admin user and uses sudo for the parts that need root:

1. Creates the service account (`octopus` by default), unless you say it already exists.
2. Installs the packages Tentacle needs.
3. Downloads and extracts Tentacle into the service user's home: `~/apps/octopus/tentacle`.
4. Tells SELinux that the Tentacle binary is a program systemd may run. SELinux gives every file a type. Files under `/home` get the type `user_home_t`, meaning personal files, and SELinux won't let systemd start a program with that type. The role changes the binary's type to `bin_t`, the type that programs in `/usr/bin` have. It does this with a permanent `semanage` rule, or with `chcon` on hosts where `semanage` can't be installed. A `chcon` change can be undone by a full SELinux relabel, so the role re-applies it on every run.
5. Writes `/etc/systemd/system/octopus-tentacle-<instance>.service` with `User=octopus`, then enables and starts it.

Every Tentacle command (`create-instance`, `new-certificate`, `configure`, `register-with`) runs as the service user. The config file, certificate and deployed applications all belong to that user, and `/etc/octopus` is never touched.

The resulting layout:

```
/home/octopus/apps/octopus/
  tentacle/                     Tentacle binaries
  instances/default/Tentacle.config
  applications/default/         deployed packages
  downloads/                    cached archives
```

## Repo layout

```
ansible.cfg                     turns on SSH pipelining (needed, see below)
site.yml                        example playbook
inventory/hosts.ini             example inventory
inventory/group_vars/tentacles.yml   example settings
inventory/group_vars/windows_tentacles.yml   example Windows connection settings
collections/requirements.yml    the ansible.windows collection, for Windows hosts
roles/octopus_tentacle/
  defaults/main.yml             every setting with comments
  tasks/linux.yml               Linux: preflight, user, install, selinux, configure, register, firewall, service
  tasks/windows.yml             Windows: tasks/windows/*.yml
  templates/octopus-tentacle.service.j2
```

## Which ansible-core version to use

Ansible needs Python on each target, and newer ansible-core releases have dropped old Python versions. From the [ansible-core support matrix](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html#ansible-core-support-matrix):

| ansible-core | Target Python supported | Controller Python |
|---|---|---|
| 2.16 | 2.7, 3.6 to 3.12 | 3.10 to 3.12 |
| 2.17 | 3.7 to 3.12 | 3.10 to 3.12 |
| 2.20 | 3.9 to 3.14 | 3.12 to 3.14 |

RHEL 7 only has Python 2.7 out of the box, so **2.16 is the newest ansible-core that can manage it**. 2.16 is end of life, but it's the only option for RHEL 7. Run `python3 --version` (or `/usr/libexec/platform-python --version` on RHEL 8) on your other hosts to see where they fall. If you manage a mix of RHEL 7 and newer hosts from one controller, use 2.16 for everything.

## Testing from a Mac or Linux machine (no Red Hat licence needed)

`ansible-core` is open source and installs with pip. AAP is a licensed product that runs the same playbooks, so a test from a laptop exercises the same role code AAP will run. The steps below use macOS with Homebrew; on Linux, install Python 3.10 to 3.12 from your package manager instead.

**1. Install ansible-core 2.16 in a virtual environment.** It needs Python 3.10, 3.11 or 3.12 on the controller.

```bash
brew install python@3.12            # if you don't already have it
python3.12 -m venv ~/venvs/ansible216
source ~/venvs/ansible216/bin/activate
pip install "ansible-core>=2.16,<2.17"
ansible --version
```

**2. Make sure you can SSH to the target host with a key.**

```bash
ssh-copy-id <user>@<host>
ssh <user>@<host> sudo -v           # confirms the user can sudo
```

**3. Fill in the inventory and settings.**

- `inventory/hosts.ini`: set `ansible_host` to each host's address and `ansible_user` to the SSH login.
- `inventory/group_vars/tentacles.yml`: set the server URL, server thumbprint, space, environments and roles.

**4. Check Ansible can reach the hosts.**

```bash
cd ansible-role-octopus-tentacle
ansible tentacles -m ping
```

**5. Run it.** `-K` prompts for the sudo password. Leave it off if the SSH user has passwordless sudo. You can pass the API key on the command line to keep it out of files:

```bash
ansible-playbook site.yml -K -e tentacle_api_key=API-XXXXXXXXXXXX
```

To install without registering, just leave `tentacle_api_key` empty.

**6. Check the result on a host.**

```bash
systemctl status octopus-tentacle-default
ps -ef | grep [T]entacle            # first column should be octopus, not root
sudo ss -tlnp | grep 10933
```

The machine should also show as healthy in Octopus under **Infrastructure > Deployment Targets**.

**7. Run it a second time.** The recap should say `changed=0`. That proves the role is idempotent, so AAP can run it on a schedule to keep machines in line.

## Running it from AAP

1. Create a **Project** in AAP that points at this repo, or at your own fork of it.
2. Create an **Inventory** with your RHEL hosts in a group called `tentacles`. Put the settings from `group_vars/tentacles.yml` on the group's variables.
3. Create a **Machine credential** with the SSH key and **privilege escalation method: sudo**.
4. Create a **Job Template** that runs `site.yml` with that inventory and credential.
5. Pass `tentacle_api_key` as an extra variable for now. Later you can swap it for a custom credential type without changing the role, because the role only reads the one variable.

Two things to check on the AAP side:

- The execution environment's ansible-core version. RHEL 7 targets need 2.16 (see the table above).
- The `pipelining = True` setting in `ansible.cfg`. AAP should pick it up from the project root. If tasks fail with "Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user", set `ANSIBLE_PIPELINING=True` on the job template instead.

## Settings you will most likely change

All settings are in `roles/octopus_tentacle/defaults/main.yml`, with comments.

| Variable | Default | Notes |
|---|---|---|
| `tentacle_user` | `octopus` | The account Tentacle runs as. Can't be root. |
| `tentacle_create_user` | `true` | Set to `false` for an existing LDAP or AD account. |
| `tentacle_version` | empty | Empty means use `tentacle_version_by_os_major`. Set it per host to pin a version. |
| `tentacle_version_by_os_major` | 7: `8.1.2007`, 8 to 10: `latest` | Per-OS defaults. |
| `tentacle_instance_name` | `default` | Use a different name and port for each extra instance on the same host. |
| `tentacle_listen_port` | `10933` | |
| `tentacle_server_thumbprint` | (required) | Thumbprint of the Octopus Server certificate. |
| `tentacle_server_url`, `tentacle_api_key` | empty | Registration only happens when both are set. |
| `tentacle_registration_type` | `target` | `target` or `worker`. |
| `tentacle_environments`, `tentacle_roles` | `[]` | Required for a target. |
| `tentacle_worker_pools` | `[]` | Required for a worker. |
| `tentacle_tenants`, `tentacle_tenant_tags`, `tentacle_tenanted_deployment_participation` | empty | Optional, for tenanted targets. |
| `tentacle_public_hostname` | empty | Force the address Octopus uses to call the Tentacle. Usually set per host. |
| `tentacle_public_hostname_source` | `auto` | How to pick the address when it isn't forced. See below. |
| `tentacle_public_hostname_suffix` | empty | Domain added with the `hostname` or `inventory` source, for example `.corp.example.com`. |
| `tentacle_download_url` | empty | Point at an internal mirror. |
| `tentacle_archive_local_path` | empty | Push a `.tar.gz` from the controller, for hosts with no internet. |
| `tentacle_manage_firewall` | `false` | Opens the port in firewalld. |

## The address Octopus uses to reach each Tentacle

A listening Tentacle waits for the Octopus Server to call it, so the server needs an address it can resolve and route to. That depends on your DNS setup, which the role can't know. You can force the value, or let the role pick one with `tentacle_public_hostname_source`:

| Source | Uses | Good when |
|---|---|---|
| `auto` (default) | The FQDN if the host reports a dotted name, otherwise its primary IPv4 address | You don't know how DNS is set up yet. |
| `fqdn` | The name the host reports as fully qualified | Hosts are registered in DNS under their own names. |
| `hostname` + suffix | Short hostname plus `tentacle_public_hostname_suffix` | DNS follows a pattern such as `<host>.corp.example.com`. |
| `inventory` + suffix | The inventory name plus the suffix | The inventory names match DNS. |
| `ansible_host` | Whatever Ansible connected to | The Octopus Server sits on the same network as AAP. |
| `ip` | The primary IPv4 address | There's no DNS for these hosts, or IPs are fixed. |

For a single machine that doesn't fit the pattern, set `tentacle_public_hostname` on its line in `hosts.ini`. That always wins.

The role prints the address it chose on every run. If you change the address later, the next run registers the machine again with `--force`, which updates the existing machine in Octopus rather than adding a second one.

If you're not sure which to use, ask whoever manages DNS which of these the Octopus Server can resolve.

## Windows hosts

The role detects Windows hosts and runs a separate set of tasks for them. Shared settings work the same way on both: `tentacle_user`, `tentacle_listen_port`, `tentacle_server_thumbprint`, the registration settings and the public hostname settings.

### What it does on each Windows host

Ansible connects as an administrator over WinRM or SSH. It doesn't use sudo or `become` on Windows.

1. Creates a local account (`octopus` by default) unless `tentacle_create_user` is `false`. The account isn't added to any admin group, and the role warns if it finds the account in Administrators.
2. Optionally installs PowerShell 7 (`tentacle_windows_install_pwsh: true`).
3. Downloads the Tentacle MSI and installs it into `C:\Program Files\Octopus Deploy\Tentacle`.
4. Creates a named instance (`Tentacle` by default) with its home folder at `C:\Octopus`, and sets the port, application folder and trust.
5. Grants the service account the rights from the Octopus [Tentacle permissions](https://octopus.com/docs/infrastructure/deployment-targets/tentacle/windows/tentacle-permissions) and [specific user account](https://octopus.com/docs/infrastructure/deployment-targets/tentacle/windows/running-tentacle-under-a-specific-user-account) pages:
   - **Log on as a service.**
   - **Full control of the home folder** (`C:\Octopus`), where Tentacle keeps its config, logs and temporary data.
   - **Read access to `HKLM\Software\Octopus\Tentacle`**, which tells Tentacle where its config lives.
   - **Read access to the certificate's private key**, but only if the certificate is in the Local Machine certificate store. If Tentacle keeps it in `Tentacle.config`, this step does nothing.
   - **Start, stop and query rights on its own service.**
6. Registers the machine, if a server URL and API key are set.
7. Installs the Windows service, sets it to run as the service account, and starts it.

Your deployments may need more rights than these, for example to change IIS or connect to SQL Server. Grant those separately.

### Which Tentacle build

Tentacle for Windows comes in two builds. Pick one with `tentacle_windows_flavour`:

| Value | MSI | Needs |
|---|---|---|
| `net8` (default) | `Octopus.Tentacle.<version>-net8.0-windows-win-x64.msi` | Nothing extra |
| `netfx` | `Octopus.Tentacle.<version>-x64.msi` | .NET Framework 4.8 or later. The role checks this and stops if it's missing. |

The role checks which build is already installed before it changes anything. The modern .NET build ships its own runtime (`coreclr.dll` in the Tentacle folder) and the .NET Framework build doesn't. If the installed build doesn't match `tentacle_windows_flavour`, the role stops and says which build it found. It never switches builds on its own, because on a server with several instances that would change the runtime under all of them.

To switch a host to the other build, uninstall the Tentacle program first (Apps & Features, or `Win32_Product`), then run the role with the new `tentacle_windows_flavour`. The instance, its home folder, the service account and the Octopus registration all stay in place.

When Octopus ships a build for a newer .NET version, add it to `tentacle_windows_msi_url_templates` under a new name (for example `net10`) and set `tentacle_windows_flavour` to that name. The role treats every modern .NET build as one family, so moving from `net8` to a newer one is an ordinary upgrade and the build check doesn't block it.

`tentacle_windows_version` defaults to `12.3.4629`. As on Linux, a pinned version lets the role handle upgrades. `latest` uses the Octopus download permalink and only installs when Tentacle is missing.

### Adding an instance to a server that already runs Tentacle

Tentacle supports several instances on one server, and the role can add one alongside an existing instance without touching it. Give the new instance its own values for these four settings:

| Setting | Example | Why |
|---|---|---|
| `tentacle_windows_instance_name` | `Ansible` | The default, `Tentacle`, is usually the existing instance. |
| `tentacle_windows_home_dir` | `C:\OctopusAnsible` | The role reuses any `Tentacle.config` it finds in this folder, so sharing a folder would reconfigure the existing instance. |
| `tentacle_listen_port` | `10934` | The existing instance usually listens on 10933. |
| `tentacle_machine_name` | `<host>-ansible` | Registration uses `--force`, which overwrites a machine with the same name in that space. |

All instances share the same Tentacle program files, so an upgrade affects all of them. When the role finds other instances, it lists them and stops rather than install a different version. Set `tentacle_windows_version` (and `tentacle_windows_flavour`) to match what's installed, or set `tentacle_windows_allow_shared_upgrade: true` to upgrade all instances on purpose. The role also stops if something else is already listening on `tentacle_listen_port`.

When you pass Windows paths with `-e` from a Mac or Linux shell, quote them (`-e 'tentacle_windows_home_dir=C:\OctopusAnsible'`). Unquoted, the shell removes the backslash, and the role stops with a message saying the path isn't a full path.

### Windows versions

The Octopus docs list Windows Server 2012 to 2025 for Tentacle. ansible-core only supports Windows versions that Microsoft still supports, which leaves Windows Server 2016, 2019, 2022 and 2025. The role warns if it finds anything older than 2016.

### PowerShell

Ansible manages Windows through the built-in Windows PowerShell 5.1, so the role needs nothing else on the host. PowerShell 7 isn't required. If your deployment steps use it, set `tentacle_windows_install_pwsh: true` and the role installs it from `tentacle_windows_pwsh_msi_url`.

### Testing a Windows host from a Mac

Use the same virtual environment as for the Linux hosts, then:

```bash
pip install pywinrm
ansible-galaxy collection install -r collections/requirements.yml
```

Add the host under `[windows_tentacles]` in `inventory/hosts.ini`. To connect with a domain account, write `ansible_user` as a UPN (`administrator@corp.example.com`) or double the backslash (`CORP\\Administrator`), because the INI format drops a single backslash. The connection settings in `inventory/group_vars/windows_tentacles.yml` use WinRM on port 5985 with NTLM. Then check Ansible can connect (`-k` prompts for the Windows password):

```bash
ansible windows_tentacles -m ansible.windows.win_ping -k
```

Run the playbook, passing the password for the new service account and your API key:

```bash
ansible-playbook site.yml -k --limit windows_tentacles \
  -e tentacle_windows_service_password='A-Strong-Passw0rd!' \
  -e tentacle_api_key=API-XXXXXXXXXXXX
```

Check the result on the host. `StartName` should be `.\octopus`, not `LocalSystem`:

```powershell
Get-CimInstance Win32_Service | Where-Object PathName -like '*Tentacle.exe*' | Select-Object Name, State, StartName
```

Run it a second time and look for `changed=0`.

### Windows settings

| Variable | Default | Notes |
|---|---|---|
| `tentacle_user` | `octopus` | A local account name, `DOMAIN\user`, or a gMSA (`DOMAIN\name$`). Set `tentacle_create_user: false` for domain accounts and gMSAs. |
| `tentacle_windows_service_password` | (required) | Not needed for a gMSA. After changing it, run once with `tentacle_windows_service_password_changed: true`. |
| `tentacle_windows_flavour` | `net8` | `net8` or `netfx`. |
| `tentacle_windows_version` | `12.3.4629` | Or `latest`. |
| `tentacle_windows_msi_url` | empty | An internal mirror URL. |
| `tentacle_windows_msi_local_path` | empty | Push an MSI from the controller. |
| `tentacle_windows_instance_name` | `Tentacle` | Use a different name to add an instance next to an existing one. |
| `tentacle_windows_allow_shared_upgrade` | `false` | Allow the role to upgrade Tentacle binaries that other instances also use. |
| `tentacle_windows_home_dir` | `C:\Octopus` | |
| `tentacle_windows_install_dir` | `C:\Program Files\Octopus Deploy\Tentacle` | |
| `tentacle_manage_firewall` | `false` | Adds an inbound Windows Firewall rule for the port. |
| `tentacle_windows_install_pwsh` | `false` | Installs PowerShell 7. |

## How reruns and upgrades behave

- **Version.** When you pin a version, the role runs `Tentacle version` and reinstalls if it doesn't match. It stops the service first and restarts it afterwards. With `latest`, the role only installs when Tentacle is missing and never upgrades on its own, so pin a version if you want Ansible to control upgrades.
- **Configuration.** `configure --reset-trust` wipes the trust list, so running it every time would be wrong. The role stores a hash of the port, application path and thumbprint next to `Tentacle.config` and only reconfigures when the hash changes.
- **Registration.** Same idea. The role hashes the registration settings and registers again (with `--force`) only when they change. The API key isn't part of the hash, so rotating the key doesn't trigger a re-registration. Set `tentacle_force_register: true` to register anyway.
- **Secrets.** The registration task runs with `no_log`, so the API key never appears in output or AAP job logs. If registration fails, the role prints Tentacle's own error message instead. `Tentacle.config` holds the Tentacle's private key and is set to mode 0600.

## Not covered yet

- Polling mode. The role installs Listening Tentacles only.
- Uninstall. To remove an instance, stop and disable the unit, delete the unit file and delete `~/apps/octopus/instances/<instance>`.
- Upgrades from the Octopus portal. These may not work with this layout, depending on the OS and Tentacle version, so they aren't supported here. To upgrade, pin the new version with `tentacle_version` and run the playbook again.

## What has been tested

- `ansible-lint` passes on the `production` profile.
- **RHEL 7** with ansible-core 2.16 from macOS: a full install, registration as a deployment target and a green health check in Octopus. A rerun with a changed public hostname re-registered the existing machine in place. That host had no working package repos, so it used the `chcon` fallback for SELinux.
- **RHEL 8, 9 and 10** with ansible-core 2.16 from macOS: a full install of Tentacle `latest` (12.2.4560 at the time), registration as a deployment target and a green health check in Octopus. SELinux used the persistent `semanage` rule. A second run reported `changed=0` on each.
- A local test run against a mock Tentacle binary covered the logic paths: version mismatch, a failed registration, worker registration, every public hostname source and the preflight checks.
- **Windows Server 2019** (domain member) with ansible-core 2.16 from macOS over WinRM: both the `net8` and `netfx` builds, including an in-place upgrade from Tentacle 9.2.4285 to 12.3.4629. The service runs as the local non-admin `.\octopus` account, and the Octopus health check reports `Local Administrator: False`. Registration as a deployment target, a green health check, and `changed=0` on a second run, for each build.
- **Windows Server 2025** (domain member) with ansible-core 2.16 from macOS over WinRM: a new instance added next to two existing ones (a listening and a polling instance) that share Tentacle 12.2.4530. The role found the other instances, refused to upgrade the shared binaries until the version was pinned, then created the new instance with its own home folder, port and machine name without touching the others, which still passed their own health checks afterwards. The firewall step added a rule for the new port. The new instance runs as the local non-admin `.\octopus` account, the Octopus health check passed, and a second run reported `changed=0`.
- **Windows Server 2016 and 2022** and the PowerShell 7 option haven't been tested on real hosts yet.

## Support

If you find a bug or encounter a problem with this role, please raise an [issue](https://github.com/OctopusDeployLabs/ansible-role-octopus-tentacle/issues). Octopus Support doesn't cover this project, so please don't open a support ticket for it.

## License

This project is licensed under the [Apache License 2.0](LICENSE).
