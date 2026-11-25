# Ansible Role: Unbound

Configure OpenBSD's base-system Unbound as a recursive, caching DNS resolver,
with optional local forward/reverse records and unbound-adblock integration.
Only OpenBSD is supported. FreeBSD support may be added later.

**The role replaces the main `unbound.conf`, enables the service and restarts
it when configuration changes.** Existing configuration is not merged. Review
your listener, access-control and DNSSEC requirements before applying it;
configuration not exposed by this role is not preserved.

## Requirements

- ansible-core 2.18 or later on the control machine.
- An OpenBSD target with a Python interpreter supported by your Ansible version.
  Set `ansible_python_interpreter` in inventory if necessary.
- Gather distribution facts before applying the role. Normal fact gathering is
  sufficient.
- Connect as root, or configure privilege escalation capable of managing system
  files, users, packages, services and other users' crontabs.
- For adblocking, install the optional control-machine dependencies below and
  provide internet access from the target for the script and upstream blocklists.
  The randomised cron schedule requires OpenBSD 6.7 or newer.

Unbound and doas are already part of OpenBSD's base system. Python installation
and initial administrative access are outside this role's scope. Adblocking
also installs the `ripgrep` and `mawk` packages; check their availability for
your target architecture.

OpenBSD runtime verification is outstanding. Syntax, template and isolated
logic checks do not establish successful installation or idempotence on a real
OpenBSD host, and no particular OpenBSD release has yet been runtime-verified.

## Installation

Add the role's Git source to your Ansible repository's requirements file:

```yaml
# roles/requirements.yml
roles:
  - name: sjktje.unbound
    src: https://github.com/sjktje/ansible-role-unbound.git
    scm: git
    version: main
```

```sh
ansible-galaxy role install -r roles/requirements.yml -p roles
```

Use a reviewed commit or release tag instead of `main` for reproducible
deployments.

For local development, put a symlink named `roles/sjktje.unbound` in your
Ansible repository pointing to this role's checkout, and ensure `roles` is on
Ansible's role search path.

### Optional Dependencies

Adblocking requires `sjktje.doas` 2.0.0 and the `community.general` collection,
which provides Ansible's doas become plugin. From this role's checkout:

```sh
ansible-galaxy role install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

The doas role's `rules` entry point manages the named adblock privileges. It
does not run the main doas role or establish general administrative access.
The dependency is included only when adblocking is enabled or when removing
its existing managed privileges after disabling it. Fresh resolver-only hosts
do not require these dependencies unless the playbook itself uses doas
privilege escalation.

## Usage

### Resolver Only

This example connects as root. Adapt the connection and escalation settings to
your environment rather than enabling root SSH access solely for this role.

```yaml
---
- name: Configure network DNS
  hosts: dns_servers
  remote_user: root
  gather_facts: true
  roles:
    - role: sjktje.unbound
      vars:
        unbound_interfaces:
          - 127.0.0.1
          - 192.0.2.53
        unbound_access_control:
          - "127.0.0.0/8 allow"
          - "192.0.2.0/24 allow"
        unbound_local_zone: '"example.org." transparent'
        unbound_local_data:
          - "host.example.org. IN A 192.0.2.10"
        unbound_local_data_ptr:
          - "192.0.2.10 host.example.org."
```

Replace the documentation addresses with addresses assigned to your target
and client network. Without interface overrides, Unbound uses its default
loopback listeners; this role disables IPv6 by default. LAN service requires
both a suitable listener and an explicit client ACL. Firewall policy and
clients' resolver settings are managed separately.

### Adblocking

Install the optional dependencies and add these variables to the play or
relevant inventory group:

```yaml
unbound_adblock: true
unbound_remote_control_enable: "yes"
unbound_remote_control_interface: 127.0.0.1
unbound_remote_control_use_cert: "yes"
```

The role downloads the stable unbound-adblock 0.5 script from
<https://geoghegan.ca/pub/unbound-adblock/0.5/unbound-adblock.sh>, verifying a
pinned SHA-256 checksum. Upgrading the script requires an explicit URL and
checksum update in the download task. The script fetches its own upstream
blocklists; those lists are not pinned by this role.

## Role Variables

Defaults are defined in [defaults/main.yml](defaults/main.yml). Unbound switch
values are quoted strings (`"yes"` or `"no"`); `unbound_adblock` is a YAML
boolean. List overrides replace the defaults rather than extending them.

### Resolver And Local Records

| Variable | Type | Default | Purpose |
| --- | --- | --- | --- |
| `unbound_conf` | string | `/var/unbound/etc/unbound.conf` | Destination of the managed configuration. |
| `unbound_checkconf_bin` | string | `/usr/sbin/unbound-checkconf` | Configuration validator executable. |
| `unbound_interfaces` | list of strings | `[]` | Listener addresses; empty leaves Unbound's default. |
| `unbound_do_ip6` | string | `"no"` | Enable IPv6 DNS transport. |
| `unbound_access_control` | list of strings | `["127.0.0.0/8 allow"]` | Complete `access-control` directive values. |
| `unbound_hide_identity` | string | `"yes"` | Hide server identity. |
| `unbound_hide_version` | string | `"yes"` | Hide server version. |
| `unbound_local_zone` | string | `""` | One complete `local-zone` value, including domain quoting; empty omits it. |
| `unbound_local_data` | list of strings | `[]` | Forward record values, such as `host.example.org. IN A 192.0.2.10`. |
| `unbound_local_data_ptr` | list of strings | `[]` | Reverse record values, such as `192.0.2.10 host.example.org.`. |

The template adds outer quotes to forward/reverse record values. Standard
`localhost` records are supplied by Unbound; no private-domain localhost
records are added. A `transparent` local zone permits normal resolution of
names not supplied by your local data.

Changing `unbound_conf` does not reconfigure the service or upstream script to
use a different path. Keep the standard path unless you separately arrange
for all consumers to use the alternative configuration.

### Remote Control

| Variable | Type | Default | Purpose |
| --- | --- | --- | --- |
| `unbound_remote_control_enable` | string | `"no"` | Enable administrative control; required for adblocking. |
| `unbound_remote_control_use_cert` | string | `"no"` | Authenticate TCP control connections with certificates. |
| `unbound_remote_control_interface` | string | `127.0.0.1` | Control address or absolute Unix-socket path. |

For adblocking, unauthenticated control is accepted only on loopback or a local
Unix socket. Loopback without certificates allows other local users to control
Unbound. Prefer certificates or a suitably permissioned local socket on
multi-user hosts. Do not expose unauthenticated control to a network, even
when adblocking is disabled and its validation is skipped.

For TCP certificate authentication, the role runs `unbound-control-setup`
before validating the configuration. It creates local server/client
credentials in `/var/unbound/etc`; no public certificate authority is needed.
The task skips setup when `unbound_control.pem` exists, so it does not repair
a partially missing credential set. Unix sockets do not use these certificates.
Control TLS is separate from DNS-over-TLS, which this role does not configure.

### Adblock Settings

| Variable | Type | Default | Purpose |
| --- | --- | --- | --- |
| `unbound_adblock` | boolean | `false` | Enable installation, generation and scheduled updates. |
| `unbound_whitelist_file` | string | `/var/unbound/db/whitelist.txt` | Whitelist destination, owned by `_adblock`. |
| `unbound_whitelist` | list of strings | See below | Upstream RPZ whitelist entries, one per line. |

The default whitelist is:

```yaml
unbound_whitelist:
  - googleadservices.com
  - www.googleadservices.com
  - '*.googleadservices.com'
```

Set `unbound_whitelist: []` if no exceptions are wanted. Custom whitelist
directories must already exist and be accessible to `_adblock`.

## Operation

On initial adblock installation, the role creates an empty, writable RPZ file,
configures and starts Unbound, applies pending handlers, then runs the script
as `_adblock`. The upstream script requires a running resolver and working
remote control. Until generation succeeds, an empty zone provides no blocking.
It also requires both log files to exist and be writable, including the
`.log.0.gz` placeholder that it replaces during rotation.

Generation runs when the script or whitelist changes, or the zone is empty or
missing. Otherwise, a daily cron job updates blocklists at a random time
between midnight and 02:00. Both execution paths use explicit script and
command search paths. Routine zone updates use `unbound-control` without
restarting the service.

Configuration changes restart Unbound, briefly interrupting DNS service;
a reload does not reliably apply remote-control changes. With adblocking
enabled, the role flushes handlers before generation. This also runs any
pending handlers from earlier roles in the play.

### Disabling Adblocking

Apply the role with `unbound_adblock: false` to remove its RPZ configuration,
named cron job and `unbound adblock` doas ruleset. Keep `sjktje.doas` available
for that run if the managed ruleset exists. Cleanup honours `doas_config_file`
when overridden and only includes the dependency if its named block is found.
Other cron jobs and doas rules remain untouched, including old unmarked rules
that must be reviewed and removed manually.

The `_adblock` account, downloaded script, whitelist, RPZ file, logs and packages
are retained. Remote-control settings are not automatically disabled. Removing
the role from a playbook does not perform cleanup; apply the disabled state
first. Cleanup does not terminate an update process that is already running.

## Verification Status

Available checks have covered role syntax on ansible-core 2.18 and 2.22,
template rendering, metadata, and isolated adblock decision/cleanup scenarios.

Before production deployment, use an expendable OpenBSD host to verify initial
installation, a second idempotent run, configuration validation, adblock
generation and whitelist changes, missing-zone recovery, disabling adblocking,
and check-mode behaviour. In particular, a first-run check-mode pass is not
guaranteed when users, script files or certificate files do not yet exist.

## Acknowledgements

Adblocking uses Jordan Geoghegan's
[unbound-adblock](https://geoghegan.ca/unbound-adblock.html) project. Its
[OpenBSD installation guide](https://geoghegan.ca/pub/unbound-adblock/0.5/install/openbsd.txt)
documents the account, file, privilege and scheduling requirements implemented
by this role. The downloaded script retains its own copyright and licence
notice; this role's licence does not replace it.

## Licence

This role is distributed under the [BSD-3-Clause licence](LICENSE).

## Author

Svante Kvarnström ([sjktje](https://github.com/sjktje)).
