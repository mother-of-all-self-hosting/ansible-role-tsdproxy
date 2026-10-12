<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://ansible.readthedocs.io/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems:
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## What the suite can and cannot tell you

Read this before changing anything under `molecule/`.

TSDProxy's entire purpose is to put a service on a [Tailscale](https://tailscale.com/) tailnet. Doing that for real needs a Tailscale pre-authentication key and a reachable control plane, neither of which CI can be given: an authkey is a credential that joins a network, and a test that used one would enrol throwaway machines into somebody's real tailnet on every run. So there is a hard ceiling on what this suite can assert, and it is worth being precise about where it sits.

### Where the ceiling is

TSDProxy discovers a service by watching Docker for containers labelled `tsdproxy.enable=true`. Everything up to *registering the resulting node with the control plane* works without a tailnet: given a labelled container, TSDProxy finds it through the socket, auto-detects its target URL on the container network, creates the proxy and starts a `tsnet` server for it. Only the final step - authenticating to `controlUrl` with the authkey - needs the real thing.

TSDProxy 1.x panicked at that last step when the authkey was unusable, so the suite used to leave labelled containers out altogether. TSDProxy 2.x fails it gracefully: the proxy's status becomes `Error` and the process carries on. **So the scenario now starts one labelled service** (`traefik/whoami`, on a network of its own, publishing no port) and asserts that TSDProxy discovered it and resolved its target to the service's address on the shared network. The proxy's status is shown in the summary, but not asserted on.

To keep that final step local, the scenario points `tsdproxy_config_tailscale_providers_default_controlurl` at a port nothing listens on, and sets `TS_NO_LOGS_NO_SUPPORT` so that the `tsnet` node does not upload its logs to `log.tailscale.com`. The fake authkey never leaves the machine.

Wiring up a local [Headscale](https://headscale.net/) as the control plane would move the ceiling, at the cost of turning this into a two-application scenario. That has not been attempted.

### What a green run does prove

Here is a synopsis of what a successful run proves. Check the scenario itself for details about what are exactly checked.

- The role's configuration file is the one TSDProxy reads, and that TSDProxy accepts it.
- `tsdproxy_container_http_port` reaches the process
- `tsdproxy_configuration_extension_yaml` survives into the running process
- `tsdproxy_container_additional_volumes` renders as a real `--mount`
- TSDProxy really has the Docker socket
- TSDProxy can reach a service the way this role sets it up
- The image's Docker `HEALTHCHECK` agrees
- The running TSDProxy is the version `defaults/main.yml` pins
- The role's other plumbing lands

### Things worth knowing before editing

- `tsdproxy_gid` must be the gid of the group that owns the Docker socket
- TSDProxy configures itself from its YAML file, not from the environment
- The image's built-in `HEALTHCHECK` has to wait for its 60-second interval before it reports anything

## Scenarios

Currently these testing scenarios are available:

### `default`

A single TSDProxy instance with one labelled service to discover (but no tailnet to register it with), configured entirely through the role's variables and with every value chosen to be distinguishable from what TSDProxy would pick for itself.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default
```
