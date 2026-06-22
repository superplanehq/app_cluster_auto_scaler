# Cluster Autoscaler

[![Launch in SuperPlane](https://superplane.com/badges/launch-in-superplane.svg)](https://app.superplane.com/install?repo=github.com/superplanehq/app_cluster-auto-scaler-hetzner-grafana-slack)

Autoscale a Hetzner Cloud worker fleet from SuperPlane. The canvas reacts to Grafana CPU alerts, provisions and bootstraps new worker nodes, records active nodes in memory, and periodically scales down quiet clusters.

Built with [SuperPlane](https://superplane.com).

## How it works

1. **Scale up** - a Grafana `HighCPU` alert starts the provisioning lane.
2. **Check capacity** - the canvas reads the active worker ledger and stops if the cluster is already at the configured max.
3. **Create worker** - Hetzner creates a new `cpx11` server and cloud-init installs Docker.
4. **Bootstrap workload** - SSH starts a sample workload container and verifies it is healthy.
5. **Remember node** - the worker is saved in the `cluster_nodes` memory namespace.
6. **Scale down** - every 10 minutes, the canvas checks average CPU and removes the newest worker when the cluster is above the configured minimum.

## Prerequisites

- [SuperPlane](https://superplane.com) account
- Grafana integration with a `HighCPU` alert
- Hetzner integration with permissions to create and delete servers
- Slack integration for scale notifications
- SSH private key secret named `cluster-deploy` with key `private-key`
- Hetzner SSH key named `cluster-deploy`

## Setup notes

The template ships with example defaults:

- Min nodes: `2`
- Max nodes: `4`
- Server type: `cpx11`
- Location: `fsn1`
- Workload image: `nginxdemos/hello:latest`

After installing, bind your Grafana, Hetzner, and Slack integrations, then update the Slack channel, Hetzner location/server settings, SSH key names, and workload bootstrap commands to match your environment.

## License

MIT
