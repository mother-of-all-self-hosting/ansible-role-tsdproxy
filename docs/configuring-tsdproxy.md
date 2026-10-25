<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Thomas Miceli
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2025 MASH project contributors
SPDX-FileCopyrightText: 2026 Bergrübe

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up TSDProxy

This is an [Ansible](https://www.ansible.com/) role which installs [TSDProxy](https://almeidapaulopt.github.io/tsdproxy/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

TSDProxy is an application that automatically creates a proxy to virtual addresses in your [Tailscale](https://tailscale.com/) network.

See the project's [documentation](https://almeidapaulopt.github.io/tsdproxy/docs/) to learn what TSDProxy does and why it might be useful to you.

## Prerequisites

To run a TSDProxy instance it is necessary to prepare a Tailscale instance.

If you are looking for an Ansible role for [Headscale](https://headscale.net/), you can check out [ansible-role-headscale](https://github.com/mother-of-all-self-hosting/ansible-role-headscale) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

## Adjusting the playbook configuration

To enable TSDProxy with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# tsdproxy                                                             #
#                                                                      #
########################################################################

tsdproxy_enabled: true

########################################################################
#                                                                      #
# /tsdproxy                                                            #
#                                                                      #
########################################################################
```

### Configure authentication

TSDProxy supports AuthKey and OAuth authentication methods with Tailscale.

To authenticate with AuthKey, add the following configuration to your `vars.yml` file:

```yaml
tsdproxy_config_tailscale_providers_default_authkey: YOUR_AUTHKEY_FOR_TAILSCALE_HERE

# AuthKey file can be specified instead
# If this is defined, `tsdproxy_config_tailscale_providers_default_authkey` is ignored.
tsdproxy_config_tailscale_providers_default_authkeyfile: YOUR_AUTHKEY_FILE_FOR_TAILSCALE_HERE
```

To authenticate with OAuth, you need to adjust `tsdproxy_config_tailscale_providers_default_*` variables.

Refer to [this page](https://almeidapaulopt.github.io/tsdproxy/docs/advanced/tailscale/) on the official documentation for details.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `tsdproxy_environment_variables_additional_variables` variable

Refer to [the official documentation](https://almeidapaulopt.github.io/tsdproxy/docs/v3/advanced/environment-variables/) for a complete list of TSDProxy's config options that you can put in `tsdproxy_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, TSDProxy becomes available internally to other services on the same network.

### Adding a new service

This proxy creates for each service a own machine in the Tailscale network, without creating each time a sidecar container.

To add a new service, you have to make sure that the service and proxy are in a same container network. You can do this by adding the proxy to the network of the service:

```yaml
tsdproxy_container_additional_networks_custom:
  - YOUR-SERVICE-NETWORK
```

The next step is to add the service to the proxy. There are two ways of doing so; one with container labels and the other with a proxy list.

#### Connecting a service to the proxy via container labels

To connect a service with container labels, add the following items to the container (with a `SERVICE_container_labels_additional_labels_custom` variable for example, if other Ansible roles by the MASH are utilized):

```yaml
- tsdproxy.enable=true
- tsdproxy.port.1=443/https:8080/http
```

`tsdproxy.port.1` makes the service available via HTTPS on port 443 of its Tailscale machine and proxies to port 8080 of the container. Without it, TSDProxy uses the first port the container exposes.

The following labels are optional. Refer to the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/docker/) for more information, e.g. about options such as `tailscale_funnel` or `no_tlsvalidate`.

```yaml
- tsdproxy.name=my-service
- tsdproxy.autodetect=false
- tsdproxy.proxyprovider=providername
- tsdproxy.ephemeral=false
- tsdproxy.port.2=80/http->https://my-service.your-tailnet.ts.net
```

#### Connecting a service to the proxy via a proxy list

An alternative way is to use proxy lists. Refer to the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/lists/) for details.

First, You need to define the lists with the `tsdproxy_config_lists` variable by adding the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
tsdproxy_config_lists:
  critical:
    filename: /config/critical.yaml
    defaultProxyProvider: default
    defaultProxyAccessLog: true
```

Then, put the list file into the config folder (`tsdproxy_config_path`), which is mounted at `/config` in the container. This is possible manually or by using [AUX-Files](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/components/auxiliary/README.md). The list file has to exist before TSDProxy starts, or TSDProxy refuses to start. A list file looks like this:

```yaml
nas:
  ports:
    443/https:
      targets:
        - http://192.168.1.2:5000
```

You can also use `tsdproxy_configuration_extension_yaml` to override individual variables from the default configuration.

### Access to the container socket

TSDProxy discovers the services it should proxy by watching the container daemon, so it needs to be able to talk to it. By default the role bind-mounts `/var/run/docker.sock` into the container read-only. Point `tsdproxy_docker_endpoint` somewhere else to change that, and set `tsdproxy_docker_endpoint_is_unix_socket` to `false` if you are using a TCP endpoint (in which case nothing is mounted).

**`tsdproxy_gid` has to be the group that owns the socket.** The container runs as `tsdproxy_uid:tsdproxy_gid`, and the socket is usually `root:docker` mode `0660`, so a `tsdproxy_gid` that is not the socket's group cannot open it.

>[!NOTE]
> If you would rather not hand a container the real socket, put something like [ansible-role-container-socket-proxy](https://github.com/mother-of-all-self-hosting/ansible-role-container-socket-proxy) in front of it and point `tsdproxy_docker_endpoint` at that instead.

### The dashboard

TSDProxy serves a dashboard and an API on `tsdproxy_container_http_port` (default `8080`) inside the container.

Since TSDProxy v2.2.0 the dashboard and the API require authentication: a Tailscale identity, or an API key (`apiKey` / `apiKeyFile`, which you can set via `tsdproxy_configuration_extension_yaml`). Note, however, that **when TSDProxy runs in a container it always lets requests from loopback and private (RFC 1918) addresses in** (`adminAllowLocalhost`), with full admin rights. Any container that shares a network with TSDProxy, and anything that can reach the published port, can therefore use the dashboard and the API. Publish the port on a loopback address only, if at all.

## Troubleshooting

### Upgrading from TSDProxy v1

This role installs TSDProxy v2, whose configuration file format is incompatible with v1. The role takes care of the file it renders, but anything you add to it yourself has to follow the new format, and so do your services' labels and proxy lists.

Refer to the [upstream upgrade guide](https://almeidapaulopt.github.io/tsdproxy/docs/upgrading/from-v1/) for the details.

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu tsdproxy` (or how you/your playbook named the service, e.g. `mash-tsdproxy`).

#### Increase logging verbosity

If you want to increase the verbosity, add the following configuration to your `vars.yml` file:

```yaml
tsdproxy_config_log_level: debug
```
