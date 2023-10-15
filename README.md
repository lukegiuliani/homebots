README.md

# Getting started TL;DR:

* Duplicate all the files in the `/secrets/` tree with files ending with `.secret` instead of `.txt` with the appropriate secret. 
* Duplicate `.env.example` to `.env`, replacing vars as needed

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
        compose.pihole.yml
        .env
        pihole.conf
```

# Secrets

We are using docker secrets. There are stub files broken out by service in the secrets folder, you need to dupe each `.txt` file, change the suffix to `.secret` and put in a more secure password. This is so we can keep track of the secrets required, without having the actual secrets in the repo. 

TODO: A convenience script to help with this? 

# Service specific readme

## Prometheus / Grafana

Based of (this awesome source)[https://github.com/docker/awesome-compose/blob/master/prometheus-grafana/compose.yaml], modified to use docker secrets and tailscale for grafana. 

### Importing dashboards

The easiest way to do this is to import them from inside grafana, configure them how you like, then go and copy the resulting json to a {template}.json file in `prometheus-grafana/grafana/provisioning/dashboards`

## Tailscale

This needs testing on a proper linux machine. Hard to see on a mac