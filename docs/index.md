---
title: SDC Docker User Guide
markdown2extras: tables, code-friendly, cuddled-lists, link-patterns
markdown2linkpatternsfile: link-patterns.txt
---

# Docker on SmartDataCenter User Guide

Welcome to Docker on SmarDataCenter. The Docker Engine for SDC is currently in alpha and under heavy development. The current focus is stabilization and filling out support for *running* Docker containers.

This document is meant as a user guide for those getting started using a Docker on SmartDataCenter service, e.g. the beta service in Joyent's public cloud (details below). If you are interested in running or developing the Docker Engine for SDC, see the
[sdc-docker README](https://github.com/joyent/sdc-docker/blob/master/README.md).

The Docker Engine for SmartDataCenter treats the entire data center as a single Docker host. Each container is a SmartOS LX-branded zone. Benefits:

- [Zones](http://en.wikipedia.org/wiki/Solaris_Containers) have a
  proven security track record for isolation.
- LX-branded zones provide the abilty to run Linux binaries in SmartOS
  zones, meaning you can use Docker images without modification *and*
  without the overhead of hardware-based virtualization.
- [Overlay network](http://en.wikipedia.org/wiki/Overlay_network)
  support (VXLAN) means you have a private network between all your containers,
  across servers.

And the full stack is open source: [sdc-docker](https://github.com/joyent/sdc-docker),
[SmartDataCenter](https://github.com/joyent/sdc),
and [SmartOS](https://github.com/joyent/smartos-live).


**A note for users of Joyent's public cloud:** Joyent is hosting a beta of their Docker service. Please sign up at https://www.joyent.com/lp/preview and read on for the settings for the standard Docker client.


## Current Status

SDC Docker is currently in alpha and under heavy development. Current focus
is on the stabilization and filling out support for *running* Docker containers.
Support for building images (`docker build`) on SDC Docker is forthcoming.
Please [report issues](https://github.com/joyent/sdc-docker/issues), give us
feedback or discuss on [#joyent IRC on freenode.net](irc://freenode.net/#joyent).


## Table of Contents

- [User Guide](./index.md)
- [Troubleshooting](./troubleshooting.md)
- [Divergence from Docker Inc. `docker`](./divergence.md)


# Getting Started

*This section will use the current Joyent-hosted SDC Docker beta for examples.
Note that the same basic instructions hold for any sdc-docker standup.*

SDC Docker is all about using the `docker` CLI. So all that is required is
to setup an account with the SmartDataCenter cloud and to configure your
`docker` client to use the appropriate auth information.

1. Install `docker`.
2. Setup an account with the SmartDataCenter (in this case the [Joyent
   Cloud](https://my.joyent.com)).
3. Run the 'sdc-docker-setup.sh' script to setup your Docker client auth.


## 1. Install `docker`

If you have docker version 1.4.1 or higher then you can move on to the next
step. *Note: The minimum docker client version might be raised to 1.5.0.*

    $ docker --version
    Docker version 1.4.1, build 5bc2ff8

Otherwise, please follow [Docker's own installation
instructions](https://docs.docker.com/installation/#installation).


## 2. Setup an SDC account

If you have a Joyent Cloud account *and* are setup to use [its
CLI](https://apidocs.joyent.com/cloudapi/#getting-started) in any of the
Joyent Cloud data centers, then you can move on to the next step. To test that
you are setup you can run `sdc-getaccount`:

    $ sdc-getaccount
    {
      "id": "....387c",
      "login": "jill",
      "email": "jill@example.com",
      "companyName": "Acme",
      "firstName": "Jill",
      ...
    }

Otherwise you need to:

- [Create](https://my.joyent.com/landing/signup/) or [sign
  in](https://my.joyent.com) to your Joyent Cloud account, and
- [Add an SSH public key to your account.](https://my.joyent.com/main/#!/account)


If you are able to, use an SSH public key you already have on your machine
or create a new one via something like:

    # Create an SSH public/private key pair of type "RSA", with no passphrase
    # (you can add a passphrase if you like, drop the '-P ""').
    ssh-keygen -t rsa -C "my-sdc-docker-key" -P "" -f ~/.ssh/sdc-docker.id_rsa -b 4096

This will create:

    ~/.ssh/sdc-docker.id_rsa      # your private key file
    ~/.ssh/sdc-docker.id_rsa.pub  # your public key file

It is the *public* key that you want to upload via the "Import Public Key"
button on [your account page](https://my.joyent.com/main/#!/account).

For more details on account setup and adding an SSH key, see [the Joyent
Cloud Getting Started documentation](https://docs.joyent.com/jpc/getting-started-with-your-joyent-cloud-account).


You can now move on to the next section. **Optionally** you can now setup the
SDC command line tools (called node-smartdc). However, that is not required
to run `docker`. See the [CloudAPI Getting Started
documentation](https://apidocs.joyent.com/cloudapi/#getting-started).

(For those using an SDC Docker standup other than the Joyent beta service,
see the [User Management](https://docs.joyent.com/sdc7/user-management) operator
guide docs.)


## 3. sdc-docker-setup

Now that you have access to the SmartDataCenter, we will setup authentication
to the Docker host. SDC Docker uses Docker's TLS authentication. This section
will show you how to create a TLS client certificate from the SSH key you
created in the previous section. Then we'll configure `docker` to send that
client certificate to identify requests as coming from you.

We have a 'sdc-docker-setup.sh' script to help with this:

    curl -O https://raw.githubusercontent.com/joyent/sdc-docker/master/tools/sdc-docker-setup.sh
    sh sdc-docker-setup.sh <CLOUDAPI> <ACCOUNT> ~/.ssh/<PRIVATE_KEY_FILE>

For example, if you created an account with the "jill" login name and a key
file "~/.ssh/sdc-docker.id_rsa" as in the previous section, then

    sh sdc-docker-setup.sh https://us-east-3b.api.joyent.com jill ~/.ssh/sdc-docker.id_rsa

That should output something like the following:

    Setting up for SDC Docker using:
        Cloud API:       https://us-east-3b.api.joyent.com
        Account:         jill
        SSH private key: /Users/localuser/.ssh/sdc-docker.id_rsa

    Verifying credentials.
    Credentials are valid.
    Generating client certificate from SSH private key.
    writing RSA key
    Wrote certificate files to /Users/localuser/.sdc/docker/jill
    Get Docker host endpoint from cloudapi.
    Docker service endpoint is: tcp://<Docker API endpoint>

    * * *
    Successfully setup for SDC Docker. Set your environment as follows:

        export DOCKER_CERT_PATH=/Users/localuser/.sdc/docker/jill
        export DOCKER_HOST=tcp://<Docker API endpoint>
        alias docker="docker --tls"

    Then you should be able to run 'docker info' and you see your account
    name 'SDCAccount' in the output.

Run those `export` and `alias` commands in your shell and you should now
be able to run `docker`:

    $ docker info
    Containers: 0
    Images: 0
    Storage Driver: sdc
    SDCAccount: jill
    Execution Driver: sdc-0.1.0
    Operating System: SmartDataCenter
    Name: us-east-3b


# TODO

- where to go if hit troubles
- usage examples
