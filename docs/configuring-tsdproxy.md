<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Thomas Miceli
SPDX-FileCopyrightText: 2024, 2025 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up TSDProxy

This is an [Ansible](https://www.ansible.com/) role which installs [TSDProxy](https://www.tsdproxy.com) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

TSDProxy is an open-source message-broker software which supports Advanced Message Queuing Protocol (AMQP), and Streaming Text Oriented Messaging Protocol (STOMP), MQ Telemetry Transport (MQTT) and other protocols with plug-ins.

See the project's [documentation](https://www.tsdproxy.com/docs) to learn what TSDProxy does and why it might be useful to you.

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

### Set default user's password

You also need to specify a password for the default user. To do so, add the following configuration to your `vars.yml` file. The value can be generated with `pwgen -s 64 1` or in another way.

```yaml
tsdproxy_environment_variables_tsdproxy_default_pass: YOUR_SECRET_KEY_HERE
```

>[!NOTE]
> The default user's name is `user` specified with `tsdproxy_environment_variables_tsdproxy_default_user`.

### Exposing the instance (optional)

By default, the TSDProxy instance is not exposed externally, as it is mainly intended to be used in the internal network, connected to other services.

To expose it to the internet, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
tsdproxy_hostname: "example.com"

tsdproxy_container_labels_traefik_enabled: true
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

**Note**: hosting TSDProxy under a subpath (by configuring the `tsdproxy_path_prefix` variable) does not seem to be possible due to TSDProxy's technical limitations.

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

After running the command for installation, TSDProxy becomes available internally to other services on the same network. If the service is exposed to the internet, it becomes available at the specified hostname like `https://example.com`.

Refer to [this page](https://www.tsdproxy.com/docs/use-tsdproxy) on the documentation about its usage.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu tsdproxy` (or how you/your playbook named the service, e.g. `mash-tsdproxy`).
