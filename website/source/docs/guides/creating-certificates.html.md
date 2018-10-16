---
layout: "docs"
page_title: "Creating and Configuring Certificates"
sidebar_current: "docs-guides-creating-certificates"
description: |-
  Learn how to create certificates for Consul.
---

# Creating and Configuring Certificates

Correctly configuring TLS can be a complex process, especially given the wide
range of deployment methodologies. This guide will provide you with a
production ready TLS configuration.

~> Note that while Consul's TLS configuration will be production ready, key
   management and rotation is a complex subject not covered by this guide.
   [Vault][vault] is the suggested solution for key generation and management.

The first step to configuring TLS for Consul is generating certificates. In
order to prevent unauthorized cluster access, Consul requires all certificates
be signed by the same Certificate Authority (CA). This should be a _private_ CA
and not a public one like [Let's Encrypt][letsencrypt] as any certificate
signed by this CA will be allowed to communicate with the cluster.

~> Consul certificates may be signed by intermediate CAs as long as the root CA
   is the same. Append all intermediate CAs to the `cert_file`.


### Reference Material

- [Encryption](/docs/agent/encryption.html)

### Estimated Time to Complete

10 minutes

### Prerequisites

This guide assumes you have Consul 1.4 (or newer) in your PATH.

## Creating Certificates

### Step 1: Create a Certificate Authority

There are a variety of tools for managing your own CA, [like the PKI secret
backend in Vault][vault-pki], but for the sake of simplicity this guide will
use Consul's builtin TLS helpers:

```shell
$ consul tls ca create
==> Saved consul-ca.pem
==> Saved consul-ca-key.pem
```

The CA certificate (`consul-ca.pem`) contains the public key necessary to
validate Consul certificates and therefore must be distributed to every node
that requires access.

~> The CA key (`consul-ca-key.pem`) will be used to sign certificates for Consul
nodes and must be kept private.

### Step 2: Create Server Certificates

Create a server certificate for datacenter `dc1` and domain `consul`, if your
datacenter or domain is different please use the appropriate flags:

```shell
$ consul tls cert create -server
==> Using consul-ca.pem and consul-ca-key.pem
==> Saved consul-server-dc1-0.pem
==> Saved consul-server-dc1-0-key.pem
```

In order to authenticate Consul servers, server certificates are provided with a
special certificate - one that contains `server.dc1.consul` in the `Subject
Alternative Name`. If you enable `verify_server_hostname`, only agents that can
provide such certificate are allowed to boot as a server. An attacker could
otherwise compromise a Consul agent and restart the agent as a server in order
to get access to all the data in your cluster! This is why server certificates
are special, and only servers should have them provisioned.

### Step 3: Create Client Certificates

Create a client certificate:

```shell
$ consul tls cert create -client
==> Using consul-ca.pem and consul-ca-key.pem
==> Saved consul-client-0.pem
==> Saved consul-client-0-key.pem
```

Client certificates are also signed by your CA, but they do not have that
special `Subject Alternative Name` which means that if `verify_server_hostname`
is enabled, they cannot start as a server.

## Configuring Agents

By now you created the certificates you are going to need to enable TLS in your cluster. The next steps will demonstrate how to configure TLS. However take this with a grain of salt because actually turning on TLS might not be that straight forward depending on your setup. For example in a cluster that is being used in production you want to turn on TLS step by step to ensure there is no downtime like described [here][guide].

### Step 1: Setup Consul servers with certificates

The following files need to be copied to your Consul servers:

* `consul-ca.pem`: CA public certificate.
* `consul-server-dc1-0.pem`: Consul server node public certificate for the `dc1` datacenter.
* `consul-server-dc1-0-key.pem`: Consul server node private key for the `dc1` datacenter.

Here is an example agent TLS configuration for Consul servers which mentions the
copied files:

```json
{
  "verify_incoming": true,
  "verify_outgoing": true,
  "verify_server_hostname": true,
  "ca_file": "consul-ca.pem",
  "cert_file": "consul-server-dc1-0.pem",
  "key_file": "consul-server-dc1-0-key.pem"
}
```

After a Consul agent restart, your servers should be only talking TLS.

### Step 2: Setup Consul clients with certificates

Now copy the following files to your Consul clients:

* `consul-ca.pem`: CA public certificate.
* `consul-client-0.pem`: Consul client node public certificate.
* `consul-client-0-key.pem`: Consul client node private key.

Here is an example agent TLS configuration for Consul agents which mentions the
copied files:

```json
{
  "verify_incoming": true,
  "verify_outgoing": true,
  "ca_file": "consul-ca.pem",
  "cert_file": "consul-client-0.pem",
  "key_file": "consul-client-0-key.pem"
}
```

After a Consul agent restart, your agents should be only talking TLS.

## Configure the Consul CLI for HTTPS

It is possible to disable the HTTP API and only allow HTTPS by setting:

```json
{
    "ports": {
        "http": -1,
        "https": 8501
    }
}
```

If your cluster is configured like that, you will need to create additional
certificates in order to be able to continue to access the API and the UI:

```shell
$ consul tls cert create -cli
==> Using consul-ca.pem and consul-ca-key.pem
==> Saved consul-cli-0.pem
==> Saved consul-cli-0-key.pem
```

If you are trying to get members of you cluster, the CLI will return an error:

```
$ consul members
Error retrieving members:
  Get http://127.0.0.1:8500/v1/agent/members?segment=_all:
  dial tcp 127.0.0.1:8500: connect: connection refused
$ consul members -http-addr="https://localhost:8501"
Error retrieving members:
  Get https://localhost:8501/v1/agent/members?segment=_all:
  x509: certificate signed by unknown authority
```

But it will work again if you provide the certificates you provided:

```
$ consul members -ca-file=consul-ca.pem -client-cert=consul-cli-0.pem \
  -client-key=consul-cli-0-key.pem -http-addr="https://localhost:8501"
  Node     Address         Status  Type    Build     Protocol  DC   Segment
  ...
```

This process can be cumbersome to type each time, so the Consul CLI also
searches environment variables for default values. Set the following
environment variables in your shell:

```shell
$ export CONSUL_HTTP_ADDR=https://localhost:8501
$ export CONSUL_CACERT=consul-ca.pem
$ export CONSUL_CLIENT_CERT=consul-cli-0.pem
$ export CONSUL_CLIENT_KEY=consul-cli-0-key.pem
```

* `CONSUL_HTTP_ADDR` is the URL of the Consul agent and sets the default for
  `-http-addr`.
* `CONSUL_CACERT` is the location of your CA certificate and sets the default
  for `-ca-file`.
* `CONSUL_CLIENT_CERT` is the location of your CLI certificate and sets the
  default for `-client-cert`.
* `CONSUL_CLIENT_KEY` is the location of your CLI key and sets the default for
  `-client-key`.

After these environment variables are correctly configured, the CLI will
respond as expected.

### Note on SANs for Server and Client Certificates

Using `localhost` and `127.0.0.1` as `Subject Alternative Names` in server
and client certificates allows tools like `curl` to be able to communicate with
Consul's HTTPS API when run on the same host. Other SANs may be added during
server/client certificates creation with `-additional-dnsname` to allow remote
HTTPS requests from other hosts.

## Summary

When you completed this guide, your Consul cluster has TLS enabled and is protected against all sorts of attacks plus you made sure only your servers are able to start as a server! A good next step would be to read about [ACLs][acl]!

[letsencrypt]: https://letsencrypt.org/
[vault]: https://www.vaultproject.io/
[vault-pki]: https://www.vaultproject.io/docs/secrets/pki/index.html
[guide]: /docs/agent/encryption.html#configuring-tls-on-an-existing-cluster
[acl]: /docs/guides/acl.html

