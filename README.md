# Getting started TL;DR:

* Duplicate `.env.example` to `.env`, replacing vars as needed. This is both in the root folder, as well as the base of each system you want to use (e.g., Tailscale)
* If using prometheus and node exporters, set a `prometheus-grafana/prometheus/targets.json` based off `targets.json.example`
* If using prometheus / grafana, set a `prometheus-grafana/secrets/grafana-admin-password.secret` based on `grafana-admin-password.txt`

# TODO:

* Sort out SSL for the line above lol. (maybe service.host.domain.com?)
* Fix `Error response from daemon: Address already in use`
* Fix (very occasional): `Error response from daemon: Pool overlaps with other one on this address space`
* Convert `pihole`s admin password to use docker secrets instead of a file. 
* Sort out the fact that there are about a million different spots for .env vars, and secrets: Move all the env vars into the root folder. Too hard otherwise. 

# Approach

So the approach we are taking is multiple compose files by chunk, and then starting it all by creating `compose.yml` variants that have the services we want. For example:

```bash
$ docker compose -f compose.all.yml up -d
```

The structure is that all the `compose.yml`s exist in the root folder (because of the relative file reference constraint of merge), with config all housed in project folders. For example:

```
/
    compose.yml
    prometheus-grafana/
        compose.prometheus-grafana.yml
        prometheus/
        grafana/
    pihole/
        .env
        compose.pihole.yml
        ...
```

# Config

There are two modes or working - local development, and then deployment. 

* Local development
    * .env files
    * Secrets are in local conf files
    * The workflow is to essentially do docker compose loops: 

```
docker compose -f compose.all.yml down --remove-orphans && docker compose -f compose.all.yml up -d
```

* Deployment:
    * .env.prod files that are amended by 1Password lookups. 
    * Secrets are taken from 1password (therefore you need to have those containers running)
    * You deploy using ansible:

```
ansible-playbook main.yml
```

Credentials:

| Item           | Dev                      | Prod                      |
|----------------|--------------------------|---------------------------|
| Tailscale Key  | .env file                | .env.prod amended by 1p   |
| Grafana login  | local .secrets file      | 1P written to .secrets    |
| Unifi pass     | unifi-poller/.env        | .env.prod amended by 1p   |


# Secrets

We are using docker secrets. There are stub files broken out by service in the secrets folder, you need to dupe each `.txt` file, change the suffix to `.secret` and put in a more secure password. This is so we can keep track of the secrets required, without having the actual secrets in the repo. 

TODO: A convenience script to help with this? 

# Service specific Readme

## Prometheus / Grafana

Based of [this awesome source](https://github.com/docker/awesome-compose/blob/master/prometheus-grafana/compose.yaml), modified to use docker secrets and tailscale for grafana. 

### Importing dashboards

The easiest way to do this is to import them from inside grafana, configure them how you like, then go and copy the resulting json to a {template}.json file in `prometheus-grafana/grafana/provisioning/dashboards`

## Tailscale

A few things.

* The docker compose files floating around the internet aren't great. Best to migrate from the official doco's command to compose using https://www.composerize.com/ or similar. That's how I got to this one (pro tip: `network_mode: host`) 
* If you remove a machine or expire a key, and you are using persistent storage, you need to kill the state as well as replace the key for it to work. Otherwise (I'm guessing) it conflicts between the fact that the state files were made with an old key, and you're now trying to use a new one. 
* Incidentally, wiping the state folder is also the best way to get rid of the incrementing suffixes problem if you keep cycling for some reason. 
* The fact that the state folder is owned by root seems scary, but actually isn't. 

# Deployment

## 1Password, Ansible, and git. 

Idea:

* Use ansible to clone the repo from github
* Create the files containing secrets by retrieving those secrets from 1Password, and directly writing them to the destination file. [Something like](https://medium.com/@robbytaylor/retrieving-secrets-from-1password-with-ansible-4364725c36b0):

```
- name: Pi public key
  copy:
    dest: '/home/{{ user }}/.ssh/rasberrypi.local.pub'
    content: "{{ lookup('onepassword', 'Raspberry Pi', field='public') }}"
    mode: 0640
```

* For this we need to run 1Password Connect on the localmachine. [Here's the ansible specific setup](https://github.com/1Password/ansible-onepasswordconnect-collection). And here's the [1Password doco for setting it up in the account](https://developer.1password.com/docs/connect/). 
** Essentially, download the `password-credentials.json` file into the 1password folder and `docker compose up``.


Then you'd deploy this all with a few steps:

* Install Ansible: `sudo apt-get install -y python3-pip`
* Clone the repo: `git clone https://github.com/lukegiuliani/homebots.git`. Enter the directory: `cd homebots`.
* Set up and set your credentials for 1Password connect. 
* Install requirements:  `ansible-galaxy collection install -r requirements.yml`
* Customise env vars:
    * TODO: Write this list. 
* Run the playbook: `ansible-playbook deploy.yml`

## Deployment sequence:

1. Collect secrets from 1Password. 
2. Check out / update the repo from github. 
3. Copy the relevant .env.prod files, replacing

## Docker contexts? TL;DR: No.

Can we use docker contexts?

Idea:

* Do basical machine set up essentially manually:
** Install docker
** Get rid of `systemd-resolved` on ubuntu per [pihole docker documentation](https://github.com/pi-hole/docker-pi-hole/#installing-on-ubuntu-or-fedora).

Then use docker contexts:

```bash
docker context create remote --docker "host=ssh://user@remotemachine"
```

*I suspect this isn't goint work as we are using secrets, referencing files, and those files don't exist on the destination machine, and getting them there is non-trivial.*

