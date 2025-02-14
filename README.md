Ansible configuration management for OpenUp
===========================================

Ansible allows us to repeatably deploy configuration to servers
with little or no manual steps. It should make it almost trivial
to set up a new server and move existing apps to it.

| [Best practises](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
| [Video](https://www.youtube.com/watch?v=5BhAJ4mEfZ8)
|

Avoid having one gigantic playbook that does everything. Rather have slightly
more granular playbooks. The arrangement we're trying at the moment is

- Set up users and groups - the first playbook we run on a new server
- Set up dokku - the second playbook we run on a new app server
- Apps - playbooks that install specific apps on one or more servers for one or more environments


### Inventory

Default inventory is `sandbox`.

Specify the inventory file to use e.g. `--inventory inventory/prod.yml`


### Roles

Roles are standardised configurations that can be applied to multiple servers.
e.g. we might have a Database server role, a Web server role. Right now we just
have a dokku server role.


### Playbooks

Playbooks deploy roles to specific servers.

We're also using a playbook per app to deploy that app to the right servers
with the right config for each environment needed.


### Conflict

Once something is managed by ansible, it should only be managed by ansible.
Otherwise someone will come and override your manual change on the server
when they run a playbook.

If you can't get ansible to do what you need and manually change something,
it's best to update this table to make it clear that ansible is not maintaining
that service on that server any more.

See the playbook for what it does and doesn't do for you.

| Server | Service | Managed by Ansible yet | Notes |
|--------|---------|------------------------|-------|
| hetzner1.openup.org.za | operating system users | yes | except `ubuntu` |
| pmg4-aws.openup.org.za | dokku installation | no | Initially installed using ansible but it's not clear whether running the dokku-server play will try to upgrade dokku which needs all apps stopped and rebuilt. |
| pmg4-aws.openup.org.za | operating system users | yes ||
| pmg4-aws.openup.org.za | dokku installation | yes ||
| pmg4-aws.openup.org.za | Dokku app: pmg | yes |   |


Setting up your controller (probably your work laptop)
------------------------------------------------------

Install the [bitwarden command line client](https://bitwarden.com/help/cli/) and login to your bitwarden account.

Install the necessary ansible plugins

```
ansible-galaxy install dokku_bot.ansible_dokku,v2020.1.6
ansible-galaxy install git+https://github.com/OpenUpSA/ansible-modules-bitwarden,a5b05a9da5cb3ba05ea6a32b284621b738d8f674
```


Managing admin access to servers
--------------------------------

### New admins

Add new admins to ansible's inventory

1. Add their key to the `files/ssh-keys` directory
2. Add them to the correct user list:
  - An admin that should be on all hosts should be added to `all_hosts_admins` in `users.yml` and run
  - An admin for only specific hosts should be added to the list `host_extra_admins` for the relevant hosts in `hosts.yaml`

### Run the playbooks to add them to the right servers

Add their operating system users

    ansible-playbook --limit hetzner1.openup.org.za users.yml

If you're not an admin on the server yet, authenticate with the initial superuser
credentials, e.g. `--user root --ask-pass`

Or you might need to specify an SSH key file for the initial non-root admin user:

    ansible-playbook --limit dokku123-aws.openup.org.za  -i inventory/staging.yml --user=ubuntu --become --key-file ~/.ssh/Bob.pem users.yml

Allow them to ssh as dokku for deployments

    ansible-playbook --limit dokku123-aws.openup.org.za -i inventory/staging.yml dokku-server.yml --tags dokku-ssh-keys


### Removing admin access

1. Move their username from `all_hosts_admins` to `all_hosts_remove_admins` in
   all inventory files relevant
2. Move their username from `host_extra_admins` to `host_remove_extra_admins` in
   all inventory files relevant
3. If they are not an admin on any server any more, remove their key from `files/ssh-keys`
3. Run the users.yml playbook and log the servers their account was removed from to inform the following step.
4. Remove their SSH key from dokku for each server they could access with `sudo dokku ssh-keys:remove ...username...`


### Updating an SSH key

1. Update their ssh key in their key file
2. Run the users.yml playbook wherever they have access
3. Remove their SSH key from dokku wherever they have access with `sudo dokku ssh-keys:remove ...username...`
4. Run the dokku-server.yml playbook wherever they have access


Configuring a new server
------------------------

Run the `server.yml` playbook against it.


Install dokku for a dokku server
-------------

After creating the server,

1. Add the hostname to the `dokku` group
2. Run the server setup playbook against just the new server:

```
ansible-playbook --limit dokku9.code4sa.org dokku-server.yml
```

Installing apps
---------------

Before deployig configuration, ensure your bitwarden session is active and your local bitwarden store is up to date, e.g.

    export BW_SESSION=$(bw unlock --raw)
    bw sync

Then run the playbook for the app you'd like to configure, with the particular
environment inventory you'd like to be configuring:

    ansible-playbook --inventory inventory/staging.yml apps/pmg.yml --start-at-task "Dokku app exists"

Include the `app` tag on your dokku configuration tasks to be able to just run those

    tags:
      - app

Then when running the playbook, to just update app configuration, use `--tags app`


Configure cron to email output for error alerts
-----------------------------------------------

Emails sent by the root, ubuntu and dokku users will be configured to "come from" webapps@openup.org.za.

Add `MAILTO=webapps@openup.org.za` at the top of the `crontab -e` file as one of those users. Mails from other users usually end up in spam because it's not setting a `From` header properly.

```
ansible-playbook --limit hetzner1.openup.org.za ssmtp.yml -e "ssmtp_AuthUser=apikey ssmtp_AuthPass=...secret-api-key..."
```

Note: In the above command it is possible for example to use, `ssmtp_AuthPass=$(pass show services/sendgrid.net | head -n 1)` to obtain sendgrid.net credentials managed in secret-store repository

Familiarising yourself with Ansible
-----------------------------------

### Ping all the machines

Not in the network `ping` command sense, just an ansible command that checks that you can connect to them all

In this repo, run

```
ansible all -m ping
```

The result should look something like the following for all hosts:

```
dokku5.code4sa.org | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
dokku6.code4sa.org | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
...
```


### Run an arbitrary command on just the dokkus

Run the following, note we're using `dokkus` referring to the group in `hosts.yml`, and not `all` this time.

```
ansible dokkus -a "echo hello"
```

The output should look something like

```
dokku5.code4sa.org | CHANGED | rc=0 >>
hello

dokku8.code4sa.org | CHANGED | rc=0 >>
hello

dokku7.code4sa.org | CHANGED | rc=0 >>
hello

dokku4.code4sa.org | CHANGED | rc=0 >>
hello

dokku6.code4sa.org | CHANGED | rc=0 >>
hello
```

### Checking what a playbook would do using `--check`

Run with `--check`

```
ansible-playbook --limit dokku9.code4sa.org --check dokku-server.yml
```

Note how it says skipped around each step

```
PLAY [dokkus] ******************************************************************

TASK [Gathering Facts] *********************************************************
The authenticity of host 'dokku9.code4sa.org (18.200.13.154)' can't be established.
ECDSA key fingerprint is SHA256:Dgs79LzpwVgd/q+vlXqsnlOfZTpEGHUBekNCyruTBh8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ok: [dokku9.code4sa.org]

TASK [Set the timezone for the server to be UTC] *******************************
skipping: [dokku9.code4sa.org]

TASK [Set up a unique hostname] ************************************************
changed: [dokku9.code4sa.org]

PLAY RECAP *********************************************************************
dokku9.code4sa.org         : ok=2    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

Best Practises
==============

Prefer declarative style over imperative
----------------------------------------

Prefer approaches that only make a change if needed. The `state: present` style
works this way: tasks that support this will only create something if it doesn't
exist, and will check its existence beforehand.

Name tasks accordingly, e.g _"Redis instance exists"_ instead of _"Create redis instance"
because it won't be creating it if it already exists.
