Welcome to 3rd and last part of my home server guide.

In previous two sections' I've shown how to create a bootable ISO with some important settings changes (such as custom SSH keys for login later) to provision bare bones server, and later how to configure it using Ansible.

In this section I will show how using Docker Compose you can easily provision self-hosted applications.

Each application, if it exposes any website over HTTP or HTTPS, will be accessible over port either 80 or 443 with TLS depending on how you decide to configure it). You will be able to easily secure all incoming HTTP traffic using TLS certificates. Additionally, I will show how you can enable authentication for all your hosted applications - even if that app exposes it's services without any login capabilities, or very weak ones.

For my own use, I've created multiple docker-compose files, each for single self-hosted application. It is possible to flatten the structure and contain them in single docker-compose file, however separating each file for each service grants me more flexibility and was my conscious choice.
(not all on the screen are actually used, some are leftovers from earlier tests)

![[Pasted image 20230529212123.png]]

## Traefik reverse proxy - the skeleton for backend

Traefik is a popular open-source reverse proxy and load balancer that is commonly used as a modern HTTP router and traffic orchestrator. It acts as a gateway between incoming client requests and the services running in the backend, allowing for efficient routing, load balancing, and SSL/TLS termination.

One of the key features of Traefik is its dynamic configuration and automatic discovery of services. It integrates seamlessly with container orchestration platforms like Docker enabling it to automatically detect and configure routes for services based on labels or annotations inside docker-compose files.

This is very important because it means that Traefik, as reverse proxy, can listen for incoming connections on port 80 and/or 443 and will direct traffic to particular service we're interested in. This is better than exposing application on random ports like it's very often done with Docker containers, like port 8080, 9001 etc.

In our case, routing will be based on domain and subdomain we'll be accessing.

To run Traefik, all that is needed is to create a docker-compose files on remote server along with necessary content. For docker-compose file, I took template from [official Traefik site](https://doc.traefik.io/traefik/user-guides/docker-compose/basic-example/) and changes some bits to adapt it for my scenario:

```yml
version: "3.3"
services:

  traefik:
    image: "traefik:v2.9"
    container_name: "traefik"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/opt/traefik:/etc/traefik"
    networks:
      - frontend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.entrypoints=https"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.api.middlewares=authelia@docker"
      - 'traefik.http.middlewares.authelia.forwardAuth.address=http://authelia:9091/api/verify?rd=https://${AUTH_DOMAIN}'
      - 'traefik.http.middlewares.authelia.forwardAuth.trustForwardHeader=true'
      - 'traefik.http.middlewares.authelia.forwardAuth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email'



networks:
  frontend:
    external: true

```

This configuration sets up Traefik as a reverse proxy, listening on ports 80 and 443, and enables various routing and TLS configurations using the provided labels. The Traefik service is connected to the `frontend` network, allowing it to communicate with other services on that network.

Instead of relying on default docker network, or default one provisioned by docker-compose, I rely on own, explicitly created `frontend` network for my applications. 


```yml
version: "3.3"
services:

  traefik:
	  ...
    networks:
      - frontend
	  ...
```

`networks:` section under `traefik` puts particular service, in this case, traefik in `frontend` virtual network. 

```yml
networks:
  frontend:
    external: true
```

This section makes docker-compose aware of externally-defined network (as in, beyond given docker-compose file), which is previously mentioned `frontend`. 

I also mount `/opt/traefik` path as `/etc/traefik` inside Traefik container. This is where our configuration files will reside.

As reference for configuring Traefik, I used multiple sources (I haven't saved them all), but these two should greatly help:

- [Traefik's official template](https://github.com/traefik/traefik/blob/master/traefik.sample.yml)
- [TechnoTim's example config](https://github.com/techno-tim/techno-tim.github.io/blob/master/reference_files/traefik-portainer-ssl/traefik/config.yml)


```yml
# Entry Points configuration
# ---
entryPoints:
  http:
    address: :80
    # (Optional) Redirect to HTTPS
    # ---
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: :443

```

-   Defines two entry points, `http` and `https`, representing HTTP and HTTPS connections respectively.
-   The `http` entry point listens on port 80 and includes an optional redirection configuration to redirect HTTP requests to HTTPS.
-   The `https` entry point listens on port 443

```yml
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /etc/traefik/certs/domain.cert
        keyFile: /etc/traefik/certs/domain.key

```

-   Specifies the TLS settings for Traefik.
-   Configures a default TLS store named `default`.
-   Specifies the paths to the TLS certificate (`certFile`) and private key (`keyFile`) files

This is for self-hosted TLS certificate key. For my needs this is enough as my services are (for now) not meant to be accesible outside my LAN network.  

Generating self-signed certificate for hosting is as simple as executing these commands on my WSL2:

```bash
openssl genrsa -out domain.key 2048

openssl req -new -x509 -sha256 -key domain.key -out domain.cert -days 3650
```

I will later show, how can You upload it to remote host via Ansible.

```yml
http:
    authelia:
      forwardAuth:
        address: "http://authelia:9091/api/verify?rd=https://auth.domain.local"

    default-whitelist:
      ipWhiteList:
        sourceRange:
        - "10.0.0.0/8"
        - "192.168.0.0/16"
        - "172.16.0.0/12"

```

-   Sets up additional HTTP-related configurations for Traefik.
-   Defines an authentication middleware named `authelia` for handling authentication.
-   Specifies the address of the Authelia service endpoint for authentication verification.
-   Provides a whitelist configuration (`default-whitelist`) to allow only specific IP ranges to access the services behind Traefik. In this case, these all are private IP addresses so that ONLY requests originating from local LAN network  will be considered by Traefik

This is a complete, brief overview of Traefik. Now I will move onto Authelia as SSO solution for all hosted software


## Authelia for SSO authentication

Authelia is an open-source, self-hosted authentication and authorization server. It provides Single Sign-On (SSO) and Multi-Factor Authentication (MFA) capabilities for protecting web applications and services. It can easily be integrated with Traefik, which means that we can very add login capabilities to our hosted applications - doesn't matter if particular software already has an authentication mechanism, or how strong it is.

Protecting our resources with SSO is as simple as hosting any other application. We start with running Authelia container via docker-compose file:

```yml
---
version: '3.3'


services:
  authelia:
    image: authelia/authelia
    container_name: authelia
    volumes:
      - /opt/authelia:/config
    networks:
      - frontend
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.authelia.rule=Host(`auth.domain.local`)'
      - 'traefik.http.routers.authelia.entrypoints=https'
      - 'traefik.http.routers.authelia.tls=true'
      - 'traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.domain.local' # yamllint disable-line rule:line-length
      - 'traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true'
      - 'traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email'  # yamllint disable-line rule:line-length
    expose:
      - 9091
    restart: unless-stopped
    healthcheck:
      ## In production the healthcheck section should be commented.
      disable: true
    environment:
      - TZ=Europe/Warsaw


networks:
  frontend:
    external: true 
```

The most important part of this code is in `labels` sections:

```yml
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.authelia.rule=Host(`auth.domain.local`)'
      - 'traefik.http.routers.authelia.entrypoints=https'
      - 'traefik.http.routers.authelia.tls=true'
      - 'traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.domain.local' # yamllint disable-line rule:line-length
      - 'traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true'
      - 'traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email'  # yamllint disable-line rule:line-length
```

-   `traefik.http.routers.authelia.rule=Host('auth.domain.local')`: Specifies the routing rule for requests targeting the Authelia service. It specifies that requests with the host `auth.domain.local` should be routed to the Authelia service.
- `traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.domain.local`: Specifies the address where the forward authentication service for Authelia is located. It defines the endpoint for verifying user authentication with Authelia

`authelia:9091` references to Docker internal network endpoint for authelia. Port 9091 is default for authelia, and DNS record `authelia` comes from `container_name` directive. 

```yml
container_name: authelia
```

Looking at key config components of my config file:

```yml
default_redirection_url: https://auth.domain.local
```

Specifies the default URL that Authelia should redirect to after authentication. This URL is used when no specific redirection URL is specified

```yml
authentication_backend:
  password_reset:
    disable: false
  file:
    path: '/config/users_database.yml'
    password:
      algorithm: 'argon2id'
      iterations: 3
      memory: 65536
      parallelism: 4
      key_length: 32
      salt_length: 16
```


Configures the authentication backend for Authelia. This block includes settings related to password reset and file-based authentication. It specifies the path to the users' database file and the parameters for password hashing using the Argon2id algorithm. 

Here, users' database is just single text file - enough for me for self-hosted server like mine. [Official authelia file template](https://github.com/authelia/authelia/blob/master/examples/compose/lite/authelia/users_database.yml) can be used as reference:

```yml
---
###############################################################
#                         Users Database                      #
###############################################################

# This file can be used if you do not have an LDAP set up.

# List of users
users:
  authelia:
    disabled: false
    displayname: "Authelia User"
    # Password is authelia
    password: "$6$rounds=50000$BpLnfgDsc2WD8F2q$Zis.ixdg9s/UOJYrs56b5QEZFiZECu0qZVNsIYxBaNJ7ucIL.nlxVCT5tqh8KHG8X4tlwCFm5r6NTOZZ5qRFN/"  # yamllint disable-line rule:line-length
    email: authelia@authelia.com
    groups:
      - admins
      - dev
...
```

Here, during login, we'll be able to authenticate to our exposed applications hidden behind proxy as user `authelia` using password `authelia`.

```yml
access_control:
  default_policy: one_factor
```

Defines the access control policies for Authelia. This block specifies the default policy, which determines the default behavior when no specific access policy is defined. Here I set default policy as `one_factor` meaning any endpoint is protected via password-only by default.

```yml
session:
  name: authelia_session
  secret: unsecure_session_secret
  expiration: 3600
  inactivity: 300
  domain: domain.local
  remember_me_duration: 2M
```


Configures the session settings for Authelia. This block includes properties such as the session name, secret, expiration time, inactivity timeout, domain, and remember me duration. These settings control the behavior of user sessions and session-related features.

This is also taken from official template, but I added `remember_me_duration: 2M` to not have to login too often.


```yml
notifier:
  filesystem:
    filename: /config/notification.txt
```

Specifies the notifier configuration for Authelia. Whenever you decide to reset password during login, an email which normally would be sent with password reset link will be redirected to text file instead.

Now, having it all configured and running, if we want to deploy any new application that will be hidden behind reverse proxy, and we want to be covered with SSO by Authelia, all we have to do is to add to docker-compose file for that application is a list of few labels inside label `section`.

I will show it on an example of Heimdall, a very convenient dashboard for self-hosted services:

```yml
    labels:
      # Enable traefik for that container
      - "traefik.enable=true"
      # Create routing rules
      - "traefik.http.routers.heimdall.rule=Host(`home.domain.local`)"
      - "traefik.http.services.heimdall.loadbalancer.server.port=80"
      # Enable TLS
      - "traefik.http.routers.heimdall.tls=true"
      - "traefik.http.routers.heimdall.middlewares=authelia@docker"
```

- `"traefik.enable=true"`: Enables Traefik for the container to be included in the reverse proxy configuration. This label ensures that Traefik handles the incoming requests for this container
- `"traefik.http.routers.heimdall.rule=Host('home.domain.local')"`: Creates a routing rule for the container named "heimdall". This label specifies that requests with the hostname "home.domain.local" should be routed to this container
- `"traefik.http.services.heimdall.loadbalancer.server.port=80"`: Specifies the port (80 in this case) where the container's service is running. Traefik uses this information to forward incoming requests to the correct port on the container. This is only necessary if given container exposes multiple ports, otherwise Traefik shall automatically discover exposed port.
- `"traefik.http.routers.heimdall.tls=true"`: Enables TLS (Transport Layer Security) for the container's routing rule. This label ensures that the communication between the client and the container is encrypted using SSL/TLS. If you do not use TLS, you can leave that one out
- `"traefik.http.routers.heimdall.middlewares=authelia@docker"`: Applies middleware to the container's routing rule. Middleware is used to add additional functionality or security to the routing process. In this case, the "authelia@docker" middleware is applied to the "heimdall" routing rule

It's that simple to add any new application to container: define dedicated hostname, apply TLS and add authelia as middleware. That is enough to incorporate any new software.

![[Pasted image 20230530025127.png]]

## Setting up local domain

This may depend on many circumstances, depending on how, from how many devices and even how many people are going to access your self-hosted websites.

For single device it can even be as simple as modifying `hosts` file, and pointing domain to server's static IP address in LAN.

For me an ideal solution would be my own DNS server, which resolves queries to my local domain (i.e. `domain.local`) along with wildcard subdomain support (`*.domain.local`), so that I do not need to manually add or remove any future subdomains for new applications.

Pi-hole can be setup as DNS for such scenario, even if it's main purpose is ad-blocking.

My ISP-provided router, which I do not have a freedom of swapping for another device right now, does not allow pointing to custom DNS servers. All it allows is statically mapping hostname to IP address, which sadly means that I need to manually add or remove any entry in order to have my deployed applications accessible on all my devices within home network.

![[Pasted image 20230530040655.png]]

That is "ok" for me for now, but will be looking for other solutions.


## Alternatives to subdomain-based routing

It is possible to configure Traefik to route traffic based on URL path.

To test this possibility, I created small Python app and deployed in on my server, so I could see how app hidden behind reverse proxy will see requested resources.

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        # Parse the URL and extract the route and parameters
        url_parts = urlparse(self.path)
        route = url_parts.path
        parameters = parse_qs(url_parts.query)

        # Create the HTML response
        html = f"""
        <html>
        <body>
            <h1>Request received</h1>
            <p>Route: {route}</p>
            <p>Parameters: {parameters}</p>
        </body>
        </html>
        """

        # Send the HTML response back to the client
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(html.encode('utf-8'))

def run_server(port=80):
    server_address = ('', port)
    httpd = HTTPServer(server_address, SimpleHTTPRequestHandler)
    print(f'Server running on http://localhost:{port}')
    httpd.serve_forever()

run_server(port=8000)


```

This simple server will output whatever URL path and parameters will reach it on simple HTML page.

I created simple Dockerfile to containerize it:

```Dockerfile
# Use a base Python image
FROM python:3.9

# Set the working directory inside the container
WORKDIR /app

# Copy the main.py file from the host to the container
COPY main.py .

# Install any required dependencies
# If your main.py requires any Python packages, you can install them here
# For example:
# RUN pip install package1 package2

# Set the entrypoint command to run the main.py file
CMD ["python", "main.py"]
```


And docker-compose file for deployment:

```yml
version: '3'

networks:
  frontend:
    external: true


services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      # With this setup, when a request comes to Traefik with a path like /api/app/lol/1, Traefik will strip the /api prefix before forwarding the request to the app service
      # defines the routing rule to match requests with a path prefix
      - "traefik.http.routers.app.rule=Host(`app.domain.local`) && PathPrefix(`/testapp`)"
      # associates the stripprefix middleware with the app router
      - "traefik.http.routers.app.middlewares=stripprefix,authelia@docker"
      # configures the stripprefix middleware to remove the "/testapp" prefix from the incoming request
      - "traefik.http.middlewares.stripprefix.stripprefix.prefixes=/testapp"      
      - "traefik.http.services.app.loadbalancer.server.port=8000"
      - "traefik.http.routers.app.tls=true"
    networks:
      - frontend

```

I was very pleased because at first sight it seemed to work, because app when accessed through given URL path `/testapp` on backed had seen it as it was accessed on root `/`:

```text
https://app.domain.local/testapp/foo/bar?query=value&a=b
```

![[Pasted image 20230530041510.png]]

Unfortunately it's not that easy because then I was testing it with other real applications, it created various problems, such as with fetching assets including .CSS and .JS files.

For example, during testing of `n8n` app hosted on `/n8n` path I discovered that assets such as CSS and JS don't load properly as they are queried at `/assets` path, and not `/n8n/assets` as I would expect them to be. I am not exactly sure why yet and how to fix that behavior, but as a workaround one may consider hosting these "problematic" apps under their own subdomain, such as `n8n.example.local` without path-based routing.

I've found some questions around the web about this issue such as https://stackoverflow.com/questions/58645726/vuejs-relative-resources-doesnt-work-behind-reverse-proxy however provided fix did not work in my case.

These solutions also involve modifying the core files of Vueapp, in a way that app will expect traffic on desired endpoint we choose such as `/app` instead of root `/`, however this approach is too invasive for me and I consciously do not choose it (for now).

https://community.traefik.io/t/javascript-vue-js-app-behind-traefik-v2-0/1874/7

![[Pasted image 20230527175023.png]]


![[Pasted image 20230527173601.png]]


I'd really like to know if there is any generic way to achieve this, as I see hosting applications under different URL paths cleaner that assigning each application its own subdomain. 

If you have any insight on how can this be achieved, please let me know.


As a last bit I want to share my simple Playbook for uploading core files for functionality of my setup - the domain certificate, the Authelia and Traefik configuration files.


```yml
---
  
- name: Upload traefik.yml config
  hosts: local-1
  become: true

  tasks:
    # Traefik configuration
  - name: Create "/etc/traefik" directory
    file:
      path: /opt/traefik/certs
      state: directory
      mode: 0755
      owner: root
      group: root

  - name: Upload traefik.yml static config to server
    copy: 
      src: files/traefik.yml
      dest: /opt/traefik/traefik.yml
      mode: 0644
      owner: root
      group: root

    # Certificate files config
  - name: Upload certificate file for traefik
    copy:
      src: files/domain.cert
      dest: /opt/traefik/certs/domain.cert
      mode: 0644
      owner: root
      group: root

  - name: Upload certificate KEY file for traefik
    copy:
      src: files/domain.key
      dest: /opt/traefik/certs/domain.key
      mode: 0600
      owner: root
      group: root

  ### AUTHELIA CONFIG ###

  - name: Upload AUTHELIA static config to server
    copy: 
      src: files/authelia-conf.yml
      dest: /opt/authelia/configuration.yml
      mode: 0600
      owner: root
      group: root


  - name: Upload AUTHELIA static user DB
    copy: 
      src: files/authelia-users_database.yml
      dest: /opt/authelia/users_database.yml
      mode: 0600
      owner: root
      group: root

```

This concludes this small series. I hope I've explained everything in a very clear manner. This all contains necessary information to deploy any application with support of reverse proxy and secured behind single, universal login page.



