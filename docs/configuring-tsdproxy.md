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

### Set authkey for Tailscale

You also need to set an authkey for Tailscale by adding the following configuration to your `vars.yml` file:

```yaml
tsdproxy_config_tailscale_providers_default_authkey: '' # OR

tsdproxy_config_tailscale_providers_default_authkeyfile: '' # use this to load authkey from file. If this is defined, tsdproxy_config_tailscale_providers_default_authkey is ignored
```

Alternatively, you can use an OAuth client by setting `tsdproxy_config_tailscale_providers_default_clientid` and `tsdproxy_config_tailscale_providers_default_clientsecret` (and usually `tsdproxy_config_tailscale_providers_default_tags`).

Refer to [this page](https://almeidapaulopt.github.io/tsdproxy/docs/advanced/tailscale/) on the official documentation for details.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `tsdproxy_environment_variables_additional_variables` variable

Refer to [the official documentation](https://www.tsdproxy.com/docs/configure#supported-environment-variables) for a complete list of TSDProxy's config options that you can put in `tsdproxy_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, TSDProxy becomes available internally to other services on the same network.

### Adding a new service

This proxy creates a separate Tailscale machine (node) in the Tailscale network for each service, without creating a sidecar container each time.

To add a new service, you have to make sure that the service and proxy are in a same container network. You can do this by adding the proxy to the network of the service or the other way round.

```yaml
tsdproxy_container_additional_networks_custom:
  - YOUR-SERVICE-NETWORK
```

The next step is to add the service to the proxy. There are two ways of doing so; one with container labels and the other with a Proxy list.

#### Connecting a service to the proxy via container labels

To connect a service with container labels, add the following items to the container (with a `SERVICE_container_labels_additional_labels_custom` variable for example):

```yaml
- tsdproxy.enable=true
- tsdproxy.port.1=443/https:8080/http
```

`tsdproxy.port.1` makes the service available via HTTPS on port 443 of its Tailscale machine and proxies to port 8080 of the container. Without it, TSDProxy uses the first port the container exposes.

See the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/docker/) for further labels.

#### Connecting a service to the proxy via a Proxy list

An alternative way to add a service to the proxy is to use Proxy lists. Define them with the `tsdproxy_config_lists` variable:

```yaml
tsdproxy_config_lists:
  critical:
    filename: /config/critical.yaml
    defaultProxyProvider: default
```

and put the list file into the directory for configuration files (most likely `/mash/tsdproxy/config/`, mounted at `/config` in the container). It is possible to do so manually or by using [AUX-Files](auxiliary.md). The file has to exist before TSDProxy starts.

Please read the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/lists/) for the format of the list file.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu tsdproxy` (or how you/your playbook named the service, e.g. `mash-tsdproxy`).
