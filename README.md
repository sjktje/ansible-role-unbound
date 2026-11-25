Role Name
=========

A brief description of the role goes here.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

Optional adblocking (`unbound_adblock: true`) requires the `sjktje.doas` role
and the `community.general` collection, which provides Ansible's doas become
plugin. Install these on the control machine from this repository:

```sh
ansible-galaxy role install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

The requirements file selects `sjktje.doas` 2.0.0, whose `rules` entry point
manages the adblock privileges. This entry point does not run the main doas
role. On OpenBSD, doas itself is already part of the base system. The role
is included when adblocking is enabled, or when removing its existing managed
privileges after disabling it. Fresh resolver-only installations do not require
these dependencies unless the playbook itself uses doas escalation.

For local development, a symlink named `roles/sjktje.doas` in the consuming
Ansible repository can point to your `ansible-role-doas` checkout instead of
installing the Galaxy release. That checkout must provide the same `rules`
entry point. Keep a separate `roles/doas` symlink if other callers still use
the short name. Ensure the consuming repository's `roles` directory is on
Ansible's role search path.

When adblocking is enabled, the role downloads unbound-adblock's stable 0.5
release from `https://geoghegan.ca/pub/unbound-adblock/0.5/unbound-adblock.sh`
and verifies a pinned SHA-256 checksum before installation. Script upgrades
require an explicit update to the URL and checksum in the download task.

Adblock Operation
-----------------

Adblocking requires working remote control. For example, enable control on
loopback with certificate authentication:

```yaml
unbound_adblock: true
unbound_remote_control_enable: "yes"
unbound_remote_control_interface: 127.0.0.1
unbound_remote_control_use_cert: "yes"
```

The role creates control keys before validating the configuration when TCP
certificate authentication is enabled. Unauthenticated control is accepted
only on loopback or a local Unix socket. Loopback without certificates allows
other local users to control Unbound; prefer certificates or a suitably
permissioned local socket on multi-user hosts.

The role starts Unbound with an empty RPZ file on first installation, applies
pending configuration changes, then generates the blocklist as `_adblock`.
The upstream script requires Unbound and remote control to be running before
generation. Until generation succeeds, an empty zone provides no blocking.
The script also requires both log files to exist and be writable, including
the rotated `.log.0.gz` path; it replaces that placeholder during rotation.

Generation runs when the script or whitelist changes, or the zone is empty
or missing. Otherwise, the daily cron job handles blocklist updates. Both
execution paths use an explicit script path and command search path.

Configuration changes restart Unbound, briefly interrupting DNS service;
a reload does not reliably apply remote-control changes. With adblocking
enabled, handlers are flushed before generation. This also runs any pending
handlers from earlier roles in the play. Routine blocklist updates use
`unbound-control` without restarting the service.

Disabling Adblocking
-------------------

Apply the role with `unbound_adblock: false` to remove its RPZ configuration,
named cron job and `unbound adblock` doas ruleset. Keep `sjktje.doas` available
for that run if the managed ruleset exists. The role checks for the ruleset
before including the dependency, honouring `doas_config_file` when overridden.
Other cron jobs and doas rules are left alone, including any old unmarked rules
that must be reviewed and removed manually.

The `_adblock` account, downloaded script, whitelist, RPZ file, logs and installed
packages are retained. Remote-control settings are not automatically disabled.
Removing the role from a playbook does not perform cleanup; apply the disabled
state first. Cleanup does not terminate an update process that is already running.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
