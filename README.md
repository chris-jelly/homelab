# 🏠 Homelab

## Introduction

This repo contains the configuration and documentation for my homelab k3s cluster.

I started this project as a place to try new things and learn new skills. Self-hosting quickly exposes the areas where you are rusty or unfamiliar, and it has been the most effective way for me to learn.

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

The cluster currently relies on node-local persistent storage for stateful workloads.
I would still like to add a proper NAS, but until then any state that matters needs to be backed up to object storage.

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
  <tr>
    <td>Sales Pipeline Pulse</td>
    <td>Internal Streamlit app for sales pipeline reporting backed by the warehouse database</td>
  </tr>
</table>

## Databases

<table>
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><a href="https://cloudnative-pg.io/">Airflow Postgres</a></td>
    <td>CloudNativePG-managed PostgreSQL cluster used by Apache Airflow</td>
  </tr>
  <tr>
    <td><a href="https://cloudnative-pg.io/">Warehouse Postgres</a></td>
    <td>CloudNativePG-managed PostgreSQL cluster for analytics and application read access</td>
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
    <td><a href="https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/">Cloudflare Tunnel</a></td>
    <td>Shared cloudflared edge tunnel that forwards public traffic to Traefik and Kubernetes Ingress resources</td>
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
    <td><a href="https://github.com/prometheus-operator/kube-prometheus">Kube-Prometheus-Stack</a></td>
    <td>Complete monitoring stack with Prometheus, Grafana, and Alertmanager</td>
  </tr>
</table>
