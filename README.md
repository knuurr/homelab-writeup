# Homelab server writeup.

An description of how I set up my home lab server.

I initially started with self-hosted home lab server as a way to experiment and learn with core DevOps technologies, including Docker, Ansible, cloud-init/Autoinstall and in some future kubernetes, for easier and automated provisioning. 

This writeup assumes at least some knowledge epsecially on Ansible, SSH, containers. It is more of an overview, rather than extensive walkthrough.

Entire writeup is divided into 3 parts. Each part touches different stages of deployment:

- ![Preparing hardware + automating Linux ISO creation](01%20Preparing%20hardware%20+%20automating%20customized%20ISO.md)
- ![Configuring server with Ansible](02%20Ansible.md)
- ![Running self-hosted apps with Docker-Compose, along with configuring reverse proxy + SSO for all apps](03%20Docker.md)

I've made such division because I suspect not everyone may be interested in whole walkthrough, and is just interested in part of it. Besides that, keeping it all in 1 document would result in a very long file, which would be hard to come back to later (for you) and maintain (for me).

Anyway, I hope this repo will be useful for you, epsecially if you're only entering the world of self-hosted.

All remarks related to my project welcome.




