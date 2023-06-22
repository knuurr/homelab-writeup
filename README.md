# Home Lab

Welcome to my Home Lab repository! 🏠🔬

## Description

This repository serves as a write-up and documentation for my personal Home Lab, where I explore various aspects of Linux, hosting, reverse proxy, Traefik, containers, Docker, and more. It's a playground for learning and experimenting with different technologies.

![banner](https://github.com/knuurr/homelab-writeup/assets/135069967/8168636d-ee04-41d0-8ddc-4ead3315d252)


## Features

- **Reverse Proxy**: I maintain a reverse proxy on my home server using Traefik. It allows me to efficiently manage and route incoming traffic to different services.
- **SSL with Let's Encrypt**: All my services are protected with SSL encryption, thanks to Let's Encrypt. This ensures secure communication and peace of mind.

![let's encrypt](https://github.com/knuurr/homelab-writeup/assets/135069967/928fbf1b-dca2-4c53-abb6-2ca73889dd99)


- **Containerization with Docker**: I leverage the power of Docker to encapsulate my services into containers. It provides flexibility, isolation, and easy management of individual components.
- **Single Sign-On (SSO)**: My services are protected with Single Sign-On, enhancing security and simplifying authentication for seamless user experience.
- Includes simple dashboards for system **monitoring** and **observability**.



I initially started with self-hosted home lab server as a way to experiment and learn with core DevOps technologies, including Docker, Ansible, cloud-init/Autoinstall and in some future kubernetes, for easier and automated provisioning. 

This writeup assumes at least some knowledge epsecially on Ansible, SSH, containers. It is more of an overview, rather than extensive walkthrough.

Entire writeup is divided into 3 parts. Each part touches different stages of deployment:

- ![Preparing hardware + automating Linux ISO creation](01%20Preparing%20hardware%20+%20automating%20customized%20ISO.md)
- ![Configuring server with Ansible](02%20Ansible.md)
- ![Running self-hosted apps with Docker-Compose, along with configuring reverse proxy + SSO for all apps](03%20Docker.md)

I've made such division because I suspect not everyone may be interested in whole walkthrough, and is just interested in part of it. Besides that, keeping it all in 1 document would result in a very long file, which would be hard to come back to later (for you) and maintain (for me).

Anyway, I hope this repo will be useful for you, epsecially if you're only entering the world of self-hosted.


If you're passionate about learning and experimenting with technologies like Linux, hosting, reverse proxy, containers, and Docker, join me in this exciting journey! Don't forget to star this repository and feel free to contribute by sharing your own insights, ideas, or improvements.

Let's connect and dive into the world of home lab setups together! 🚀





