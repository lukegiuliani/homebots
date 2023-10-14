README.md

# Approach

So the approach we are taking is multiple compose files by chunk, and then starting it all up [with the merge command](https://docs.docker.com/compose/multiple-compose-files/merge/). Then we can add new services easily enough. For example:

```bash
$ docker compose -f compose.yml -f compose.prometheus-grafana.yml up -d
```

The structure is that all the `compose.yml`s exist in the root folder (because of the relative file reference constraint of merge), with config all housed in project folders. For example:

```
/
    compose.yml
    compose.pihole.yml
    compose.prometheus-grafana.yml
    prometheus-grafana/
        prometheus/
        grafana/
    pihole/
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
