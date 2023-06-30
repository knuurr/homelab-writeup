# Homelab server writeup.

>**NOTE: Writeup and actual machine configuration and playbooks are currently in a state of drift away and I'm working on general update.**

An description of how I set up my home lab server.

I initially started with self-hosted home lab server as a way to experiment and learn with core DevOps technologies, including Docker, Ansible, cloud-init/Autoinstall and in some future kubernetes, for easier and automated provisioning. 

This writeup assumes at least some knowledge epsecially on Ansible, SSH, containers. It is more of an overview, rather than extensive walkthrough.

# Playbooks

Ansible playbooks for this project are available [in my repo](https://github.com/knuurr/homelab-playbooks).


# Writeup

Entire writeup is divided into 3 parts. Each part touches different stages of deployment:


- [01 Preparing hardware + automating Linux ISO creation](01_preparing_hardware.md)
- [02 Configuring machine with Ansible](02_ansible.md)
- [03 Running self-hosted apps with Docker-Compose, along with configuring reverse proxy + SSO for all apps](03_docker.md)

## Updates

>**NOTE: because writeup and actual machine configuration and playbooks are currently in a state of drift away, I want to provide updates in a form of atomic smaller guides, while I will be working on an general update.**

- [Configuring Traefik with Let's Encrypt DNS-01 Challenge using DuckDNS - no more self-signed cert!](traefik_plus_letsencrypt.md)
- [Configuring Prometheus, Grafana, and node_exporter - visualise system performance easily!](grafana_dashboard.md)

I've made such division because I suspect not everyone may be interested in whole walkthrough, and is just interested in part of it. Besides that, keeping it all in 1 document would result in a very long file, which would be hard to come back to later (for you) and maintain (for me).

Anyway, I hope this repo will be useful for you, epsecially if you're only entering the world of self-hosted.

All remarks related to my project welcome.




