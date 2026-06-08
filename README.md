# toodoo
My personal ToDo service, intended for multi-context work and MCP integration

## Why another ToDo list service?

 * Self-hosted, but with the integrations I need
 * AI features, but with self-hosted LLMs for privacy
 * MCP server for AI agents (mostly for voice assistant interface)
 * Encrypted data
 * Reasonably secure
 * Tasks can have lables to group them (e.g. private, family, job, ...)
 * Task management features (curate, auto-generate tasks, ...)
 * Simple data strucutres, easy backup and restore

## What is in this repo?

 * TooDoo Listener service (privilege seperation, does the TLS parts)
 * TooDoo TLS Signer service (privilege separation, keep TLS private key safe)
 * TooDoo Authenticator service (establish user identity, trigger IP bans)
 * TooDoo Fail2Ban helpers (configures a packet filter to temporarily ban IPs)
 * TooDoo Task service
 * TooDoo MCP server
 * TooDoo CLI client
 * build scripts for all artefacts
 * backup and restore scripts
 * update scripts
 * tests
 * documentation

## TooDoo Listener

This service accepts incoming connections for both the REST API and the MCP
service, performs the TLS handshake, and then passes the connection on to
the authenticator service. It can handle Let's Encrypt ACME certificate
renewals by watching the filesystem for new cert files. (alternatively,
we could implement ACME ourselves in the Listener, but that would require
us to be able to write cert files to disk, overwriting the old one)

The purpose is privilege separation: this service should be small, drop
almost all capabilities and should be easy to audit. TLS code has frequently
been the source of weaknesses in many services, so we are isolating it. Even
if compromised, an attacker should basically find an empty filesystem with
nothing to use for further attacks into the backend.

For further protection, we will also add a rate limit per source IP and
a total connection rate limit.

The Listener also serves the public (unprotected) parts of TooDoo, such
as the OpenAPI spec file and the web interface assets. These are
compiled into the binary, so that the service doesn't need to be able
to read from the filesystem.

## TooDoo TLS Signer

Another service that exists primarily for privilege separation and security
isolation. This service is in possession of the TLS secret key, so the
Listener service needs the help of the TLS Signer in order to perform
the TLS handshake. They talk to each other via a Unix domain socket using
gRPC.

The purpose is to keep the secret key away from the publicly exposed
Listener service. Also, this introduces an abstraction layer for how and
where the TLS key is stored, so that we could implement the Signer
in different ways later on - for example using an HSM or OpenBao.

For now, we use a simpler self-contained version that has the
key mapped into the container read-only.

## TooDoo Authenticator

This service performs authentication for incoming connections. If successful,
the connection is passed on to the Task Service, including the information
about the user that was successfully identified.

Again, the purpose is privilege separation, but also abstraction. The way
this service implements authentication does not need to be known to either
the Listener nor to the Task service. We can implement a first version
based on a hardcoded list or a local file, and later extend it with
JWT-based OpenID Connect capabilites for identity providers, or LDAP,
or whatever.

The Authenticator logs authentication attempts, both successful and otherwise.
For unsuccessful attempts, it will notify the Fail2Ban service, so that it
can act accordingly.

## TooDoo Fail2Ban

We support Fail2Ban by providing configuration data so that Fail2Ban can
do its job in blocking IPs with too many failed authentication attempts.

In this part of the project we manage the Fail2Ban config and the
necessary filters for understanding the TooDoo Authenticator log files.

## TooDoo Task Service

This implements the core functionality of this project: task management.

Here we keep track of tasks, implement CRUD functionality and additional
features such as priorities, labels, grouping, comments and more. It can
be thought of as a "poor man's ticket management system".

We include integration support for AI assistance feature, such as
automatic task classification, priority assessment, task curation,
summaries, etc. These AI features require an LLM to work, which is not
part of this project, but can be configured to enable the AI features.


