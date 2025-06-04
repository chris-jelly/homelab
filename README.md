# 🏠 Homelab

## Introduction

This repo contains the configuration and documentation for my homelab. 

I started this project primarily to have a place to try new things and learn new skills. I continue to find that the process of self-hosting applications quickly exposes you to areas that you're rusty or unfamiliar with and is in my mind the most effective way to learn.

## Hardware

### Nodes

I currently run my Homelab by way of a Raspberry Pi running the Raspberry Pi OS Lite. I am using a C4 Labs cluster case and a Raspberry Pi 5 8GB with a PoE+ hat. Later, I plan to expand this to a multi-node cluster and swapping the OS to Talos to make provisioning clusters easier.


### Storage

For the time being I have a 2TB SSD directly attached to the primary node of the cluster. Ideally I would like to get a proper NAS setup, but in the meantime may look to set up unRAID instead.

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
    <td><a href="https://github.com/sissbruecker/linkding">Linkding</a></td>
    <td>A minimal bookmark manager</td>
  <tr>
</table>


## Infrastructure

Tools and services that help run the cluster or deploy applications.

<table>
  <tr>
    <th>Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><a href="https://cert-manager.io/">Cert Manager</a></td>
    <td>Certificate management for Kubernetes </td>
  </tr>
  <tr>
    <td><a href="https://github.com/renovatebot/renovate">Renovate</a></td>
    <td>Automated dependency updates</td>
  </tr>
  <tr>
    <td><a href="https://fluxcd.io/">Flux CD</a></td>
    <td>The GitOps toolset I use</td>
  </tr>
  <tr>
    <td><a href="https://external-secrets.io/latest/">External Secrets Operator</a></td>
    <td>Used to sync my secrets from Azure Key Vaults to my cluster</td>
  </tr>
</table>