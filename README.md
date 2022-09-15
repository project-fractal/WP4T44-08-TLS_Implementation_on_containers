# WP4T44-08 TLS Implementation on containers

Be it for local or remote access, ensuring security when accessing Docker containers is a matter of great importance.

In this document, the instructions to enable TLS access on Docker are given, with set up steps, usage and troubleshooting.

This component is related to WP4 - T4.4, Security Services for the Fractal environment, and it satisfies the cybersecurity requirements for the Fractal platform and Edge environments, by ensuring secure communications with any container.

### Prerequisites

* Docker Engine
* openssl

## Getting Started
Using TLS and managing a CA is an advanced topic. Please familiarize yourself with OpenSSL, x509, and TLS before using it.

These instructions will get your Docker daemon exposed with TLS security and certificates signed by your own CA. See [Deployment](#deployment) for notes on how to deploy the component in a use case.

The configuration of the Docker daemon can be done in two ways:
* Using flags when manually starting `dockerd`
* Using a JSON configuration file. This is the preferred option, since it keeps all configurations in a sigle place.

A combination of both can be used as long as the same options is not specified both as a flag and in the JSON file. If that happens, the Docker daemon won't start and prints an error message.
This document provides guidelines to configure TLS via the JSON file.

## Deployment

### Generate CA, certificates and keys

This document presents guidelines so that the Docker daemon accepts only connections from clients that posses certificates signed with the host machine as a CA and so that the clients only connect to servers signed with that same CA.

When generating certificates with `openssl`, the library will ask several questions regarding location, company, etc. Most of them are not important, but some need to be answered with specific names or IPs, so read the guideline carefuly before executing any command.

To sign all the needed certificates, a CA is needed. As stated before, this CA is going to be generated on the host or server machine (the machine with the Docker daemon). This can be done though the following commands:

```bash
openssl genrsa -aes256 -out ca-key.pem 4096

openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
```

It is important that when asked about the `Common Name(e.g. server FQDN or YOUR name):` the answer provided is the DNS name of the host machine (e.g. the server FQDN).

With the first command, the CA key is generated and the CA itself is created on the second command. This new CA will be valid for 365 days as stated by the flag `-days`. 

Now, the server key and certificate signing request (CSR) can be generated:

```bash
openssl genrsa -out server-key.pem 4096

openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
```

$HOST should resolve the DNS name of the server machine.

Since TLS connections can be made through IP address as well as DNS name, the IP addresses need to be specified when creating the certificate. For example, to allow connections using `10.10.10.20` and `127.0.0.1`:

```bash
echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf

echo extendedKeyUsage = serverAuth >> extfile.cnf
```

Again, replace $HOST with the server's DNS name.

Now, to generate the signed certificate:

```bash
 openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```

That's all the certificates for the server, now onto the client certificates:

```bash
openssl genrsa -out key.pem 4096

openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```

To make the key suitable for client authentication, a new extensions config file needs to be created:

```bash
echo extendedKeyUsage = clientAuth > extfile-client.cnf
```

Now, to generate the signed certificate:

```bash
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
```

After generating `cert.pem` and `server-cert.pem` it is safe to remove the two certificate signing requests and extension config files:

```bash
rm -v client.csr server.csr extfile.cnf extfile-client.cnf
```

The `ca.pem`, `cert.pem` and `key.pem` need to be shared with the client machine to be able to authenticate when making remote connections.

### Daemon configuration

Now that all certificates are generated, to configure the Docker daemon with TLS the file `/etc/docker/daemon.json` needs to be created. This file should look somewhat like this:

```json
{
"tls":true,
"tlscert":"/var/docker/server-cert.pem",
"tlscacert":"/var/docker/ca.pem",
"tlskey":"/var/docker/server-key.pem",
"hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
}
```

The routes for `tlscert`, `tlscacert` and `tlskey` need to be the routes for the `ca.pem`, `server-cert.pem` and `server-key.pem` generated in the previous step of this guide.

The `hosts` field specifies which connections will the Docker daemon accept, in this case, the usual unix socket which is used by default and `tcp://0.0.0.0:2376` (i.e. a connection on any of the network interfaces of the machine on port 2376).

## Usage

To connect to the Docker daemon now the usage is as follows:

```bash
docker --tlsverify \
    --tlscacert=ca.pem \
    --tlscert=cert.pem \
    --tlskey=key.pem \
    -H=$HOST:2376

```

Replacing $HOST again with the DNS name of the server.

## Troubleshooting

On systems using `systemd` (eg. Debian and Ubuntu), Docker listens on a socket by default, this means that a host flag `-H` is always used when starting `dockerd`. If a `hosts` entry is specified in the `daemon.json`, this causes a configuration conflict and Docker fails to start.

To work around this issue, there needs to be a new file `/etc/systemd/system/docker.service.d/docker.conf` with the following contents, to remove the `-H` argument that is used when starting the daemon by default.

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```

**Note**: once this option is overriden if a `hosts` entry is not specified in the `daemon.json` or a `-H` flag when starting Docker manually, Docker fails to start.

Now to apply this new configuration:

```bash
sudo systemctl daemon-reload

sudo systemctl restart docker
```

## Additional Documentation and Acknowledgments

Helpful links:
* Generating certificates: https://docs.docker.com/engine/security/protect-access/#use-tls-https-to-protect-the-docker-daemon-socket
* Configure and Troubleshoot the Docker daemon: https://docs.docker.com/config/daemon/
