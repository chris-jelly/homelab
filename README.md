# 🏠 Homelab

## Introduction

This repo contains the configuration and documentation for my homelab. 

I started this project primarily to have a place to try new things and learn new skills. I continue to find that the process of self-hosting applications quickly exposes you to areas that you're rusty or unfamiliar with and is in my mind the most effective way to learn.

## Hardware

### Nodes

<table>
  <tr>
    <th>Node Type</th>
    <th>Hardware</th>
    <th>RAM</th>
    <th>Role</th>
  </tr>
  <tr>
    <td>Raspberry Pi 5</td>
    <td>8GB with PoE+ hat, C4 Labs cluster case</td>
    <td>8GB</td>
    <td>Control plane</td>
  </tr>
  <tr>
    <td>Laptop</td>
    <td>Various Laptops</td>
    <td>~8-16GB each</td>
    <td>Worker nodes</td>
  </tr>
</table>


### Storage

For the time being this is all running off of the local path provisioner, so whatever storage the nodes have attached.
Ideally I would like to get a proper NAS setup, need to explore options more. Until I do, any state that matters will need to be backed up to object storage.

## Applications

<table>
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><a href="https://github.com/advplyr/audiobookshelf">Audiobookshelf</a></td>
    <td>A self-hosted audiobook, podcast, and e-book server. Syncs with Calibre which in turn syncs to my Kobo</td>
  </tr>
  <tr>
    <td><a href="https://github.com/sissbruecker/linkding">Linkding</a></td>
    <td>A minimal bookmark manager</td>
  </tr>
  <tr>
    <td><a href="https://actualbudget.org/">ActualBudget</a></td>
    <td>Personal finance management application</td>
  </tr>
  <tr>
    <td><a href="https://airflow.apache.org/">Airflow</a></td>
    <td>Workflow orchestration platform with git-sync for DAG deployment</td>
  </tr>
  <tr>
    <td><a href="https://gethomepage.dev/">Homepage</a></td>
    <td>Self-hosted dashboard for monitoring homelab services and infrastructure</td>
  </tr>
</table>


## Infrastructure

Tools and services that help run the cluster or deploy applications.

<table>
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><a href="https://fluxcd.io/">Flux CD</a></td>
    <td>The GitOps toolset for continuous delivery</td>
  </tr>
  <tr>
    <td><a href="https://cert-manager.io/">Cert-Manager</a></td>
    <td>Automatic TLS certificate management with Cloudflare DNS</td>
  </tr>
  <tr>
    <td><a href="https://external-secrets.io/latest/">External Secrets Operator</a></td>
    <td>Secret synchronization from Azure Key Vault to Kubernetes</td>
  </tr>
  <tr>
    <td><a href="https://cloudnative-pg.io/">CloudNative-PG</a></td>
    <td>PostgreSQL operator for database management</td>
  </tr>
  <tr>
    <td><a href="https://github.com/renovatebot/renovate">Renovate</a></td>
    <td>Automated dependency updates via CronJob</td>
  </tr>
</table>

## Monitoring

<table>
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><a href="https://grafana.com/">Grafana</a></td>
    <td>Monitoring dashboards and visualization</td>
  </tr>
  <tr>
    <td><a href="https://prometheus.io/">Prometheus</a></td>
    <td>Metrics collection and alerting</td>
  </tr>
  <tr>
    <td><a href="https://github.com/prometheus-operator/kube-prometheus">Kube-Prometheus-Stack</a></td>
    <td>Complete monitoring stack with Prometheus, Grafana, and Alertmanager</td>
  </tr>
</table>
