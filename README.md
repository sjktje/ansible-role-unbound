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
is only included when adblocking is enabled; resolver-only use does not
require these dependencies unless the playbook itself uses doas escalation.

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
