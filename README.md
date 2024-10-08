# zenith  <!-- omit in toc -->

Zenith is a reliable, scalable and secure tunnelling HTTP(S) proxy built using industry-standard
software and protocols.

## Contents  <!-- omit in toc -->

- [What is Zenith?](#what-is-zenith)
- [Architecture](#architecture)
- [Deploying a Zenith server](#deploying-a-zenith-server)
- [Proxying a service using the Zenith Client](#proxying-a-service-using-the-zenith-client)
  - [Container image](#container-image)
  - [Python installation](#python-installation)
  - [Example: Proxing NGINX](#example-proxing-nginx)

## What is Zenith?

Zenith consists of a server and a client which collaborate to establish a secure tunnel over which
traffic can then flow to the proxied service, even if that service is behind NAT and/or a firewall.

  * Expose services that are behind NAT or a firewall as subdomains of a parent domain.
    * Exposed services only need to be bound locally, i.e. to `localhost`, on an isolated Docker network
      or within the same Podman or Kubernetes pod as the Zenith client.
  * Limit the clients that are able to connect using a token-based system.
  * Perform TLS termination for proxied services.
  * Enforce external authentication and authorization for proxied services.
  * Uses industry-standard software and protocols:
    * [OpenSSH](https://en.wikipedia.org/wiki/OpenSSH) and
      [SSH port forwarding](https://help.ubuntu.com/community/SSH/OpenSSH/PortForwarding)
      to provide secure tunnels that bridge NAT or a firewall.
    * [Kubernetes](https://kubernetes.io/) for resilient and flexible services.
    * [Kubernetes Ingress resources](https://kubernetes.io/docs/concepts/services-networking/ingress/)
      and the
      [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
      for performant, dynamic proxying.
    * (Recommended) [cert-manager](https://cert-manager.io/docs/) for managing TLS certificates.

## Architecture

Zenith has two logical components, a server and a client, each of which has subcomponents. It also
leverages the power of Kubernetes on the server-side to do most of the heavy lifting for the dynamic
proxying.

The Zenith server consists of three main components, all of which are written in
[Python](https://www.python.org/) and deployed in Kubernetes:

  * A registrar that allows subdomains to be reserved and issues single-use tokens that can
    be used to associate SSH public keys with those subdomains.
  * A locked-down SSHD server that establishes secure tunnels with the Zenith clients and posts
    the resulting service information into a store.
  * A sync component that receives updates from the store and synchronises the corresponding
    `Service`, `Endpoint` and `Ingress` resources in Kubernetes.

The Zenith client is also written in [Python](https://www.python.org/), and it is responsible for:

  * Uploading the SSH public key to the registrar using a previously issued token (the
    delivery mechanism of the token to the client is out-of-scope for Zenith).
  * Managing the SSH connection to the SSHD component of a Zenith server using the
    [OpenSSH client](https://man.openbsd.org/ssh.1).

The reservation of domains and the delivery of tokens to clients are managed by an external "broker"
that will be different for each use case. For example, [Azimuth](https://github.com/azimuth-cloud/azimuth)
is able to act as a broker for Zenith clients that are running on machines and clusters it creates.

The architecture of Zenith is described in more detail in [Zenith Architecture](./docs/architecture.md).

## Deploying a Zenith server

The only supported deployment mechanism for Zenith is to use [Helm](https://helm.sh/) to
deploy to a Kubernetes cluster. This documentation assumes that you already have a Kubernetes cluster
available for your Zenith deployment that has the
[NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) installed.

> **IMPORTANT**
>
> Before deploying Zenith, there must be a **wildcard DNS entry** pointing at the IP address
> of your Kubernetes Ingress Controller for Zenith to use - Zenith does **not** manage this.

For more detail on deploying and configuring a Zenith server, see
[Deploying and configuring a Zenith server](./docs/server.md).

## Proxying a service using the Zenith Client

The Zenith client establishes a secure tunnel with a Zenith server and ensures that traffic received
over that tunnel is forwarded to the proxied service.

The Zenith client has two subcommands - `init` and `connect`. The `init` command is responsible
for generating an SSH identity (if required) and uploading the public key to the Zenith registrar
using the token it receives from the broker - this is a one-time operation. The `connect` command
then uses the SSH identity from `init` to establish a secure tunnel over which traffic can flow
to the proxied service.

For detailed information on configuring the Zenith client, see
[Configuring the Zenith client](./docs/client.md).

### Container image

The Zenith client is made available on [GitHub Packages](https://github.com/features/packages)
as a multi-arch container image that is built for both AMD64 and ARM64:

```bash
docker run --rm ghcr.io/azimuth-cloud/zenith-client:main zenith-client --help
```

### Python installation

The Zenith client can also be installed directly from GitHub using [pip](https://pip.pypa.io/en/stable/):

```bash
# Install the Zenith client
pip install git+https://github.com/azimuth-cloud/zenith.git#subdirectory=client
# Show the Zenith client help
zenith-client --help
```

### Example: Proxing NGINX

A service need only be bound locally in order to be proxied using Zenith - it only needs to
be reachable by the Zenith client.

In this example, we start an NGINX container on an isolated Docker network and proxy it by
deploying the Zenith client onto the same network.

First, we launch an NGINX container onto an isolated Docker network. The container is launched
in such a way that it becomes a long-running, robust service (using the `--detach` and
`--restart` flags). Giving the container a name (using `--name`) means that the container can
be addressed by name by other containers on the same network using DNS, which we utilise when
connecting the Zenith client later.

```
$ docker network create zenith-test
13124561fcf532b37c65a76a648964071c1dcb158d7cf4615c88ffd4e19c20f9

$ docker run --detach --restart unless-stopped --network zenith-test --name nginx nginx
d8a1f908ec0393b86885d71f4ad1c6f05704892ae2fbc8893368fa8067d2165d
```

Next, we need to run the Zenith client `init` command. To do this, we need a token from the
Zenith registrar - this would normally be issued by a broker but in this case we perform the
role of the broker manually.

The registrar's reservation endpoint is only available within the Kubernetes cluster, but we
can use `kubectl port-forward` to access it and issue a token:

```
$ REGISTRAR_SVC="$(kubectl get svc -l app.kubernetes.io/component=registrar --no-headers | awk '{ print $1 }')"

$ kubectl port-forward svc/$REGISTRAR_SVC 0:80
Forwarding from 127.0.0.1:51485 -> 8000
Forwarding from [::1]:51485 -> 8000

$ curl -X POST -s http://localhost:51485/admin/reserve | jq
{
  "subdomain": "dgububnqpld9wv7kqac5w5aifiynbteaa3o",
  "fqdn": "dgububnqpld9wv7kqac5w5aifiynbteaa3o.apps.example.org",
  "token": "ZGd1YnVibnFwbGQ5d3Y3a3FhYzV3NWFpZml5bmJ0ZWFhM28uNWY3OWUzZmVhZWVmZDA2NjUzODJmNjljNDVjMjM2ZTE3YTRmOTNlZmNjMjg2NWJkNWI1OGRjNjBlNjNlYmNlNg==",
  "fingerprint": null,
  "fingerprints": []
}
```

Now we run the Zenith client `init` command to generate an SSH identity and upload the public
key to the Zenith registrar. We use a Docker volume to store the SSH identity so it can be passed
to the `connect` command:

```
$ docker volume create zenith-ssh
zenith-ssh

$ docker run \
    --rm \
    -v zenith-ssh:/home/zenith/.ssh \
    ghcr.io/azimuth-cloud/zenith-client:latest \
    zenith-client init \
      --ssh-identity-path /home/zenith/.ssh/id_zenith \
      --registrar-url ${zenith_registrar_url} \
      --token ZGd1YnVibnFwbGQ5d3Y3a3FhYzV3NWFpZml5bmJ0ZWFhM28uNWY3OWUzZmVhZWVmZDA2NjUzODJmNjljNDVjMjM2ZTE3YTRmOTNlZmNjMjg2NWJkNWI1OGRjNjBlNjNlYmNlNg==

[2024-04-10 13:10:19,376] zenith.client.init   [INFO    ] Generating SSH identity at /home/zenith/.ssh/id_zenith
Generating public/private rsa key pair.
Your identification has been saved in /home/zenith/.ssh/id_zenith
Your public key has been saved in /home/zenith/.ssh/id_zenith.pub
The key fingerprint is:
SHA256:c6vjwXKGMcPAoSMekUOdbtUHEvFnA2QP++LmspaMZpE zenith-key
The key's randomart image is:
+---[RSA 2048]----+
|.oo o+=*.        |
| o.= o+.=.       |
|..= +  o.=       |
|...+ o  + .      |
| .. . =.S..      |
|   E  .*.o .     |
|    + +o= .      |
|   + =o+.o       |
|  o ..ooo.       |
+----[SHA256]-----+
[2024-04-10 13:10:19,485] zenith.client.init   [INFO    ] Uploading public key to registrar at [registrar URL]
[2024-04-10 13:10:19,699] zenith.client.init   [INFO    ] Public key SHA256:c6vjwXKGMcPAoSMekUOdbtUHEvFnA2QP++LmspaMZpE uploaded successfully
```

Finally, we launch the Zenith client `connect` command onto the isolated Docker network
using the SSH identity generated in the previous step to establish the tunnel. As with NGINX,
we launch the container with the `--detach` and `--restart` flags to establish a long-running,
robust service that can recover from failures:

```
$ docker run \
    --detach \
    --restart unless-stopped \
    --network zenith-test \
    -v zenith-ssh:/home/zenith/.ssh \
    ghcr.io/azimuth-cloud/zenith-client:latest \
    zenith-client connect \
      --ssh-identity-path /home/zenith/.ssh/id_zenith \
      --server-address ${zenith_sshd_address} \
      --server-port ${zenith_sshd_port} \
      --forward-to-host nginx \
      --forward-to-port 80
acbbe2f337edfb821d504482677318d920c5e16ee144034e2b2104c56b7e4623
```

We can check the logs from the `connect` command to see that the tunnel established successfully:

```
$ docker logs acbbe2f337edfb821d504482677318d920c5e16ee144034e2b2104c56b7e4623

[2024-04-10 13:12:51,123] zenith.client.tunnel [INFO    ] Switching to uid '1001'
[2024-04-10 13:12:51,124] zenith.client.tunnel [INFO    ] Writing SSH private key data to temporary file
[2024-04-10 13:12:51,124] zenith.client.tunnel [INFO    ] Spawning SSH process
[2024-04-10 13:12:51,124] zenith.client.tunnel [INFO    ] Negotiating tunnel configuration
Warning: Permanently added '[redacted]:2222' (ED25519) to the list of known hosts.
[2024-04-10 13:12:53,073] zenith.sshd.tunnel   [INFO    ] Negotiating tunnel configuration
eyJhbGxvY2F0ZWRfcG9ydCI6IDQzMDk1LCAiYmFja2VuZF9wcm90b2NvbCI6ICJodHRwIiwgInNr
aXBfYXV0aCI6IGZhbHNlfQ==

END_CONFIGURATION
[2024-04-10 13:12:53,110] zenith.client.tunnel [INFO    ] Tunnel configured successfully
[2024-04-10 13:12:53,107] zenith.sshd.tunnel   [INFO    ] Received tunnel configuration
[2024-04-10 13:12:53,108] zenith.sshd.tunnel   [INFO    ] Allocated port for tunnel: 43095
[2024-04-10 13:12:53,164] zenith.sshd.tunnel   [INFO    ] Posted heartbeat with status 'passing'
[2024-04-10 13:13:03,185] zenith.sshd.tunnel   [INFO    ] Posted heartbeat with status 'passing'
[2024-04-10 13:13:13,207] zenith.sshd.tunnel   [INFO    ] Posted heartbeat with status 'passing'
[2024-04-10 13:13:23,226] zenith.sshd.tunnel   [INFO    ] Posted heartbeat with status 'passing'
[2024-04-10 13:13:33,248] zenith.sshd.tunnel   [INFO    ] Posted heartbeat with status 'passing'
...
```

The subdomain that is associated with the SSH key is verified, then the client and server
negotiate the tunnel configuration.

The NGINX test page will now be available at `http[s]://[subdomain].[zenith_base_domain]`.
