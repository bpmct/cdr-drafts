for coder.com/blog:

# Guide: Connect to your VS Code environment from any device

Historically, using a development environment across multiple devices would mean compromises on speed, access, or limited compatibility. With VS Code however, developing from any device is much more approachable.

Today, we're taking a look at how we can host a VS Code environment and access it from anywhere.

- VS Code (Remote SSH): Use VS Code to connect and develop on a remote machine
- code-server: Access VS Code from the web browser, runs on a remote machine

Both of these tools have their advantages and disadvantages. Here's my workflow:

- I use VS Code (Remote SSH) for near-local experience developing on a DigitalOcean cloud server
- I use code-server to connect to the DigitalOcean server from unfamiliar devices (e.g Grandma's computer) or "light" devices such as an iPad or Chromebook
- For iOS projects, I use VS Code on my MacBook but run code-server + a public tunnel when I want to develop from any other device

## Installing code-server

[code-server](https://github.com/cdr/code-server) is an open source project that hosts VS Code in the web browser. It can be installed on a remote server or locally (to make your local environment remote).

Use code-server's install script:

```bash
# dry run:
curl -fsSL https://code-server.dev/install.sh | sh -s -- --dry-run

# install:
curl -fsSL https://code-server.dev/install.sh | sh
```

Run as a docker container:

```bash
# note: running code-server as a Docker container
# will not expose your local projects and settings.
#
# to persist any dependencies installed use a custom image,
# mount volumes, or add them to the home directory

mkdir code-server && cd code-server
docker run -it --name code-server -p 127.0.0.1:8080:8080 \
  -v "$PWD:/home/coder" \
  -u "$(id -u):$(id -g)" \
  -e "DOCKER_USER=$USER" \
  codercom/code-server:latest
```

Other options in our docs: [Installing Coder](https://coder.com/docs/code-server/latest/install)

## Syncing code-server and VS Code settings (optional)

You can also configure code-server to use the same settings and extensions as VS Code.

```console
# on the machine/container with code-server installed:
vim $HOME/.config/code-server/config.yaml
```

Add the VS Code directories. On OSX, they are:

```yaml
# config.yaml

user-data-dir: "/Users/username/Library/Application Support/Code"
extensions-dir: "/Users/username/.vscode/extensions"
```

## Exposing to the internet

Once you have code-server and/or VS Code Remote installed, it's time to make them accessible remotely. Normally, this would require either port forwarding, configuring firewalls, or a VPN,

**Option 1: code-server —link**

If you'll only be connecting remotely from the web browser, you can use the `code-server --link` flag to get a public tunneled URL. Instead of authenticating with a password, you'll be prompted to log in with GitHub. Future logins with the tunneled URL will verify your GitHub account.

You can also add `link: true` to code-server's config file to launch with —link by default.

![code-server --link in Google Chrome](link.png)

Note: code-server —link will not proxy your SSH server. Other options, such as inlets can do this for you.

**Option 2:** Self-host a tunnel with [inlets PRO](http://inlets.dev) (code-server and SSH)

Inlets PRO is a self-hosted tunneling software with abstractions that make it simple to expose local services to the internet. Inlets is not free, so you'll need a license for inlets PRO. Their personal plan is $22.50/mo. Since this is also a self-hosted solution, you'll need access to a remote server and a domain nxrame.

[Alex Ellis](https://github.com/alexellis/), the founder of Inlets (and OpenFaaS), and I connected and he provided some excellent support when preparing this guide. Inlets offers a lot of flexibility when it comes to self-hosting tunnels, but also has the [inletsctl](https://github.com/inlets/inletsctl) tool to get up and running quick!

If you already have a server, you can use the [inlets documentation](https://docs.inlets.dev/#/?id=inlets-pro-reference-documentation) to install & configure inlets-pro. In this example, I use `inletsctl` to deploy an "exit" server that will cost roughly $5/mo on DigitalOcean. The inlets docs has [examples for different providers](https://docs.inlets.dev/#/tools/inletsctl?id=examples-for-specific-cloud-providers) that work great, just be sure to include the LetsEncrypt flags.

```bash
inletsctl create --provider digitalocean \
 --access-token DIGITALOCEAN_ACCESS_TOKEN \
 --region nyc1 \
 --letsencrypt-domain code.bpmct.net \
 --letsencrypt-email ben@example.com \
 --letsencrypt-issuer prod
```

Be sure to replace the domain, email, and optionally the [region](https://status.digitalocean.com/). Also include a [DigitalOcean access token](https://docs.digitalocean.com/reference/api/create-personal-access-token/) from your account.

After the server is created, `inletsctl` will return the server's IP address. Add a DNS A record for your custom domain to the IP. Inlets will also give an example command on how to start a tunnel. Change the `—upstream` flag to the where code-server is hosted (usually 127.0.0.1:8080). Also, this is where add your [inlets license](https://docs.google.com/forms/u/1/d/e/1FAIpQLScfNQr1o_Ctu_6vbMoTJ0xwZKZ3Hszu9C-8GJGWw1Fnebzz-g/formResponse).

```bash
inlets-pro http client --url "wss://[server IP]:8123" \
  --upstream "code.bpmct.net=http://127.0.0.1:8080" \

Starting HTTP client. Version 0.8.6 - 18d8e239f940d8694532e332867c25f823439cc6
Licensed to: Ben Potter <********>
Upstream:  => http://127.0.0.1:8080
Connecting to proxy
Connection established.. OK.

# access at https://code.bpmct.net

# if you're on Linux, you can generate a systemd service file
inlets-pro http client generate=systemd
```

After that, you'll be able to access your secure tunnel via your domain.

![inlets tunneled URL in Google Chrome](inlets.png)

Tunneling a local SSH server with inlets is also possible. To do this, we'll need to log in to the exit server (the one we created with inletsctl) and run a second instance of inlets in TCP mode. We'll also need to edit our local machine's SSH settings so that it can listen on another port (since the exit server already uses port 22).

```bash
# add additional local SSH port (2222)
# add a line: "Port 2222"
sudo vi /etc/ssh/sshd_config

# ssh into the exit server (DigitalOcean emails the root password)
ssh root@[SERVER_IP]

# run a TCP exit server as well
inlets-pro tcp server \
 --auto-tls \
 --auto-tls-san=[DOMAIN_NAME] \
 --control-port 8124 \
 --auto-tls-path=/tmp/tcp_certs/

# generate systemd service to keep TCP server running
inlets-pro tcp server --generate=systemd --auto-tls \
 --auto-tls-san=[DOMAIN_NAME] \
 --control-port 8124 \
 --auto-tls-path=/tmp/tcp_certs/

# create the service (ensure it has the proper flags)
sudo vim /etc/systemd/system/inlets-pro-tcp.service

# run the service on start, and start it up now
systemctl enable --now inlets-pro-tcp

# go back to our local machine
exit

# tunnel SSH as well
inlets-pro tcp client \
 --url "wss://[DOMAIN_NAME]:8124" \
 --upstream "127.0.0.1" \
 --ports 2222

# now you can ssh from any device into your local machine
# ssh -p 2222 code.bpmct.net
```

## Installing/enabling a SSH server

If you want to access your dev environment over VS Code Remote SSH, you'll need an SSH server running on you rmachine. Most operating systems have a built-in SSH server that just has to be enabled.

- [MacOS](https://support.apple.com/guide/mac-help/allow-a-remote-computer-to-access-your-mac-mchlp1066/mac)
- [Windows Server 2019 and Windows 10](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)
- Linux: You probably know what you're doing for SSH :)

## Looking forward

It's clear there are a lot of options when it comes to developing in the cloud. Wherever you are developing (local machine, managed service, or private datacenter), developers can have smooth workflows without compromises.

I'm particularly interested in we can enable these workflows while also reducing the setup/friction associated with developing on remote servers such as provisioning, managing access keys, or connecting to running services. Here are a few things we're exploring:

- a single tunnel to a server for SSH and multiple IDEs (hint. JetBrains)
- extension for tunneling ports inside code-server
- 1-click provisioning and access for dev servers
- dev teams using containers for their dev environments (local, remote, or hybrid)

Thoughts? We'd love to hear on [Twitter](https://twitter.com/coderhq) or [Slack](https://cdr.co/join-community).
