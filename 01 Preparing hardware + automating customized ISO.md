Welcome to my description on how I provision my own home server.

I've divided this journey into 3 separate writeups. Each touches different layers of provisoning and configuring server.

1) preparing installations and provisoning barebones server
2) applying configuration and preparing environment via Ansible
3) running software via Docker-Compose.

By the end of the entire journey you will have a secured, configured home server on which you can host many self-hosted software, all in a scalable way.

Main technologies we'll be using are: Autoinstall/cloud-init, Ansible, Docker/Docker Compose.


# 0: Preparations

There were few things I needed to consider for my potential hardware to make it fit my vision and needs of home server:

- It MUST have required peripherials for WiFi connectivity (in my environment cable connection is a no-go)
	- external WiFi adapters generally don't work out of the box and need additional drivers
- It should generally support Linux out of the box, without tinkering, so that I can use generic distro image, without worrying about issues with drivers etc.
- ca. 4GB of RAM minimum, best if expandable
- there must be an easy way to expand storage, because these thin clients usually have around 8GB of storage and with installing packages, container images etc. You would probably quickly run out of space

With all these requirements, in my opinion, SBC's (like Raspberry Pi) would be ideal for this kind of project. 

Although RAM is not expandable, adding storage is as easy as buying SD card with more space. There's probably no need to worry about driver issues as SBCs usually have either their official Linux images, tailroed to hardware they are built with or some kind of community which supports these devices with custom builds. This solves potential problems with provisioning devices.

![](3051885-40.jpg)

However, due to unavaibility of Pi's and high prices of used ones I had to get creative and settle for "thin clients".

"thin clients" are just generic x86 PCs wtih some kind of Intel processor, embedded small storage (usually few gigs max), some RAM. From my research,  these computers are designed to be used for VDI - streaming applications or desktop from remote location. As such they do not need to be "beefy" machines. But since they are just generic computers they are good fit to be basic home server. 

For my home lab, I went for used Dell Wyse 5060 thin client. Mine came with Wifi card AND necessary antennas for it to work, but not all these clients on marked have such setup, and if Wifi support is something you personally need, You either:

1) need to make sure device from listing you are interested in comes with Wifi card + antennas connected to it 
2) separately equip your thin client with network card + antennas

![](5060_450.jpg)

I also needed a way to output what's happening on screen of terminal during installation. This will be helpful because we first need to install Linux distro in machine. If any error arises during installation, we need to have a way to see the screen output, or even be able to enter the shell if more advanced troubleshooting i.e. checking installation logs is needed.

Because all I have is my laptop and I do not have any external display, I needed to find a way to output whatever there would be on display on my laptop's screen.

I read about solution which involves presenting such devices as "USB Camera", so that any output can be displayed using Windows' "Camera" app.

For that, I needed:
- DisplayPort to HDMI converter (mine terminal has DisplayPort)
- HDMI cable
- HDMI to USB-B converter
- (optional, only for my circumstances) USB-B to USB-C converter
	- that's only because on left side of my laptop I have USB-C port, and plugging in from right side would be inconvienient for me

![](i-retoo-adapter-displayport-do-hdmi-display-port-dp.webp)
*DisplayPort to HDMI converter*

![](10884417_800.jpg)
*HDMI cable*

![](KARTA-PRZECHWYTYWANIA-WIDEO-VIDEO-GRABBER-HDMI-USB.jpg)
*HDMI to USB-B converter*

This stack allows me to connect my thin client to my laptop and receive output signal just as it was external camera.

![](Pasted%20image%2020230530164624.png)

For this solution and USB keyboard connected to the thic client will also be needed to control 

Besides that we also need a removable media to boot our Linux image from. I went for one of the cheapest USB sticks I could find online. 

Here's a final checklist:

- 1x thin client (in my case, Dell Wyse 5060)
- (for wifi) 1x WiFi network card
- (for wifi) 1x or 2x external WIfi antennas
- (for wifi) 1x connectors from Wifi card to Wifi antennas
- 1x DisplayPort/HDMI cable to output client's
- 1x HDMI to USB-B converter
- 1x DP to HDMI converter

Now we can start provisoning our server.


# 1 Creating (semi-) unattended, automated Ubuntu Server

My first step was to create an ISO image which, ideally, would have preconfigured several mission-critical settings. This involves automating basically everything a default Ubuntu installer asks for, plus maybe one or two additional things:

- a configured default admin user, possibly with custom name, along with supplied by me SSH key, so that after installation I can simply SSH into server
- working OOTB wireless network configuration, so that after installation my server can automatically connect to my predefined home LAN network, get a private IP address and reach external internet
- instructions on how to automatically partition installation drive
- if possible, a dedicated `ansible` user for Ansible, along with configured SSH keys, so there is no need to manually create such user

I identified this as bare minimum for my ISO install. Every other configuration I may need can be handled by Ansible. later 

This way, I have my base install image with my minimal settings for remote accessibility, for working networking and prepared for using Ansible.

I can separate configuration which I consider as immutable, or very rarely changing, from configuration which may be a subject to change, and which I can modify later with Ansible or manually.


For automated base image creation, I use following technologies

- [PXEless](https://github.com/cloudymax/pxeless) - "An automated system install and image customization tool for when PXE is not an option, or is not an option yet."
- Autoinstall.yaml ([main page](https://ubuntu.com/server/docs/install/autoinstall), [reference](https://ubuntu.com/server/docs/install/autoinstall-reference)) - YAML-formatted configuration file used by Ubuntu's installer for automated installation and system configuration. It allows you to specify various settings and parameters such as partitioning, package selection, user accounts, network configuration, and more
- Cloud-init ([main page](https://cloud-init.io/), [docs](https://cloudinit.readthedocs.io/en/latest/)) - Cloud-init supports various cloud platforms and allows you to automate tasks such as setting up SSH keys, configuring networking, running scripts, creating users, and more


## Preparing Autoinstall.yaml file

The difference between cloud-init and Autoinstall is that Autoinstall Ubuntu-specific and has its own sections, while cloud-init is more generic format used by many other distros.

Autoinstall.yaml can also use cloud-init sections, I'll use it for some tasks and show it.

When downloading PXEless repo, I'd only like to point out that there is also a great example template for Autoinstall, which can be used as reference for other modifications and learning.

![](Pasted%20image%2020230528223425.png)

My entire Autoinstall.yaml is present in repo. I will only go through most important sections of my configuration, to clarify some choices I made.

NOTE: each example will have `autoinstall:` section at the top, to show how indentation inside file should look, as this may be the source of errors.

```yaml
#cloud-config
autoinstall:
  version: 1
```

This code block sets the basic structure of the autoinstall configuration file with the specified version

```yml
autoinstall:
  version: 1
  locale: en_US
  timezone: "Europe/Warsaw"
  keyboard:
    layout: "pl"
```

This code block configures the locale, timezone, and keyboard layout settings. It sets the locale to en_US, timezone to Europe/Warsaw, and keyboard layout to "pl" (Polish).

```yml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: ubuntu-server # customize
    username: admin # customize
    password: "PWD HASH" # customize
```

This code block defines the identity settings for the system. It sets the hostname to "ubuntu-server", the username to "admin", and the password to the hashed value of the desired password.

You can generate desired password with `openssl` tool:

````bash
qbus@DESKTOP-CA07ILV:~/pxeless2$ openssl passwd -6
Password:
Verifying - Password: # your input
$6$PzLvGSayURts9WDj$/r8b68VT3yCM0Ff.gm2Qk5GEMu3y1b0FRDH7YvHa6vBwkep70aX8k9OAJrGsIkA9f9GKXzWB.aE0zpoIl3xcD/
qbus@DESKTOP-CA07ILV:~/pxeless2$
````


```yml
#cloud-config
autoinstall:
  version: 1
  user-data:
    users:
      - name: ansible
        gecos: Ansible User
        sudo: "ALL=(ALL) NOPASSWD:ALL"
        groups: sudo
        lock_passwd: true
        shell: /bin/bash
        ssh_authorized_keys:
          - "SSH PUBLIC KEY"
```

This code block defines user data settings for the system. It creates a user named "ansible" with the specified details, including the ability to run sudo commands without a password. It sets the user's shell to /bin/bash and configures SSH authorized keys.

`user-data` section is actually a section dedicated for `cloud-init` settings, it can be used when Autoinstall's settings are not enough, or maybe when you already  have your own cloud-init template. This is a section to paste it to.


```yml
#cloud-config
autoinstall:
  version: 1
  network:
    version: 2
    renderer: networkd
    wifis:
      wlp2s0: # interface name
        dhcp4: no
        dhcp6: no
        addresses: [192.168.100.200/24]
        nameservers:
          addresses: [192.168.100.1, 8.8.8.8]
        access-points:
          "SSID":
            password: "PLAINTEXT-PWD"
        routes:
          - to: default
            via: 192.168.100.1
```

This code block configures the network settings for the system. In my case, it sets the network version to 2 and the renderer to `networkd`. It defines the WiFi settings for the interface named "`wlp2s0`", including static IP configuration, DNS servers, WiFi access point details, and default route. 

**However this did not work for me**. I've detected that whatever settings I supply in this section they get ignored by installer, and my machine doesn't have wifi connectivity. I'm not sure why this happens (a bug, or mistake on my end) , but I have a fix which I will show later in this section.


```yml
#cloud-config
autoinstall:
  version: 1
  storage:
    layout:
      name: direct
    grub:
      install_devices:
        - /dev/sda
    config:
      - type: partition
        id: sda-part1
        size: 256MB
        bootable: true
        wipe: true
      - type: partition
        id: sda-part2
        preserve: false
        wipe: true
      - type: format
        id: sda-part1
        fstype: fat32
      - type: format
        id: sda-part2
        fstype: ext4
      - type: mount
        id: sda-part2
        path: /
    boot-size: 256MB
```

This code block configures the storage settings for the system. It sets the storage layout to "direct" and specifies the GRUB installation devices as `/dev/sda`. The `config` section defines the storage configuration, including partitioning, formatting, and mounting. It creates two partitions (`sda-part1` and `sda-part2`) with specific sizes and file systems (fat32 and ext4, respectively). The last section sets the boot size to 256MB.

```yml
#cloud-config
autoinstall:
  version: 1
  ssh:
    install-server: true
    authorized-keys: # for user defined in "identity" section
      - SSH PUBLIC KEY 
    allow-pw: false
```

This section configures SSH settings. It enables the SSH server during the installation, sets the authorized SSH keys for admin user, and disables password-based authentication.

## "networking" section fix

I've stated that `network` section doesn’t not work as intended. because during my experiments I discovered that:
- network configuration described in autoinstall.yaml in section `network` is not to be found under `/etc/netplan` on target host
- target host will be missing `wpasupplicant` package, making it unable to operate wifi connectivity

I have a workaround for that, however this makes installation only semi-automatic.

To work around that issue, we need to add following settings to autoinstall.yaml:

```yml
#cloud-config
autoinstall:
  version: 1

  late-commands:
    - rm -f /target/etc/netplan/*
    - cp /media/10-wifi.yaml /target/etc/netplan/10-wifi.yaml

  user-data:
    runcmd:
      - netplan apply
      - ip link set wlp2s0 up
  
  
  interactive-sections:
    - network
  # - storage
```

- The `late-commands` section contains a list of commands to be executed after the installation is complete. It removes any existing netplan configuration files and copies a specific netplan configuration file (`10-wifi-lol.yaml`) to the target system's `/etc/netplan/` directory.
	- This works, because `pxeless` allows us to supply arbitrary files to installer, which during installation will be accesible under `/media` path, while target system's filesystem will be accessible under `/target` path.
	- This allows us to copy our Netplan network configuration file from our USB stick to target system during installation process.
- The `runcmd` section under `user-data` specifies a list of commands to be executed after the installation. In this case, `netplan apply` applies the network configuration copied using `late-commands`, and `ip link set wlp2s0 up` brings up the wireless interface, because I've spotted that after installation after reboot network interface wasn't up. This way, we can avoid it.
- The `interactive-sections` section defines the interactive sections during the installation process. By specifying `network`, the installer prompts for network configuration details. You can also uncomment the `storage` section to enable interactive storage configuration
	- I've discovered that it is only during network configuration screen, that Ubuntu installer will make sure that necessary packages for wifi support will be installed onto target system. This is why we need to manually go through this screen.

Tl;dr this allows to copy necessary internet config onto machine, instead of relying on autoinstall. There are also commands that ensure Netplan will apply this config and that our wifi interface is up.

We can easily create wifi config for Netplan. Actually, Autoinstall uses Netplan syntax in it's `network` section.  We can repurpose it and put in in separate config, in this case, `10-wifi.yaml`

Having such config in Autoinstall.yaml:

```yml
#cloud-config
autoinstall:
  version: 1
  network:
    version: 2
    renderer: networkd
    wifis:
      wlp2s0: # interface name
        dhcp4: no
        dhcp6: no
        addresses: [192.168.100.200/24]
        nameservers:
          addresses: [192.168.100.1, 8.8.8.8]
        access-points:
          "SSID":
            password: "PLAINTEXT-PWD"
        routes:
          - to: default
            via: 192.168.100.1
```

We just need to copy it to new file, `10-wifi.yaml` , without Autoinstall-specific sections (and with indentation adjusted, ofc):

```yml
version: 2
renderer: networkd
wifis:
  wlp2s0: # interface name
	dhcp4: no
	dhcp6: no
	addresses: [192.168.100.200/24]
	nameservers:
	  addresses: [192.168.100.1, 8.8.8.8]
	access-points:
	  "SSID":
		password: "PLAINTEXT-PWD"
	routes:
	  - to: default
		via: 192.168.100.1
```



# 2 Preparing Ubuntu ISO with PXEless

After previous section, you should have:
- a ready to use `autoinstall.yaml` config file for automated install
- Netplan wifi network configuration, for a workaround solution

We need a tool that will be able to parse our autoinstall.yaml file and generate a tailored to our needs ISO image for installation. This is where PXEless comes in.

To run PXEless, you need to have a Docker available on your machine, as this software runs as a container.

Since I use Windows machine, I needed to use a combination of WSL2 + Docker Desktop for Windows. I won't be providing an instruction on how to install set up Docker on Windows machine, because this can be found on the internet. Just make sure, that after installation Docker is usable:

```sh
docker --version
# Docker version 20.10.24, build 297e128
```

![](Pasted%20image%2020230529001236.png)

We start by cloning git repo of pxeless, and `cd` into it:

```sh
git clone https://github.com/cloudymax/pxeless.git

cd pxeless
```

For installation we need an ISO image. This guide is tailored for Ubuntu Server. I choose the latest LTS version at time of writing, this being 22.04.2.

It can be downloaded from official source [here](https://ubuntu.com/download/server) where we get a direct download link. Since we're already inside `/pxeless` directory, we can just use `wget` :

```bash
wget https://releases.ubuntu.com/22.04.2/ubuntu-22.04.2-live-server-amd64.iso
```

![](Pasted%20image%2020230529003700.png)

Inside pxeless folder, theres is extras folder. This is where we can copy Netplan config file (or any arbitrary file too), which after installation will be copied to target host.

![](Pasted%20image%2020230529003954.png)


Now, to generate ISO, all we need to do is run command below:

```bash
docker run --privileged --rm --volume "$(pwd):/data" deserializeme/pxeless -a -u autoinstall.yaml -s ubuntu-22.04.2-live-server-amd64.iso -x /data/extras
```

This will run pxeless container which will generate out ISO.

Here are few CLI options I want to describe what they do. All of them are explained on project's [github page](https://github.com/cloudymax/pxeless#command-line-options):

 - `-u autoinstall.yaml`: Specifies the path to the user-data file (`autoinstall.yaml`), which contains the configuration for the auto-installation process
 - `-s ubuntu-22.04.2-live-server-amd64.iso`: Specifies the source ISO file to use for the installation. In this case, it's the `ubuntu-22.04.2-live-server-amd64.iso` file
 - -   `-x /data/extras`: Specifies a folder (`/data/extras`) that contains additional files and folders to be copied into the root of the generated ISO. This is useful for including custom scripts or additional resources in the installation image. We use it for Netplan config.

![](Pasted%20image%2020230529005204.png)

![](Pasted%20image%2020230529005224.png)

`ubuntu-autoinstall.iso` is our image we can use for installation.

For preparing installation media, I used Rufus. Etcher did not work for me.

To quickly access location of generated ISO, you can type `explorer.exe .` inside WSL2 terminal. This will open Windows Explorer window inside pxeless folder, inside WSL2 filesystem. 

![](Pasted%20image%2020230529010500.png)

You can copy path location from there and supply it to Rufus when clicking SELECT.

There isn't much to configure. Just make sure you select proper device for installation, so you won't wipe any other attached storage. Also whether it's GPT or MBR is of course device-dependent, but it will probably be GPT.

![](Pasted%20image%2020230529010123.png)

After Rufus finished, USB stick is ready.

All we need to make sure now is that our target machine has booting from USB stick enabled AND that, if other OS is already installed, it will boot from USB first.


![](Pasted%20image%2020230530165407.png)

![](Pasted%20image%2020230530165434.png)


In the end after installation, you will have your ready to use server which you can SSH into.


