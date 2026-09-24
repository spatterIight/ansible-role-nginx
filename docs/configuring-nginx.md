<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2024 - 2025 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Nginx

This is an [Ansible](https://www.ansible.com/) role which installs [Nginx](https://nginx.org/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Nginx is a web server, which can also be used as a reverse proxy, load balancer and HTTP cache.

See the project's [documentation](https://nginx.org/en/docs/) to learn what Nginx does and why it might be useful to you.

By default, this role configures Nginx to serve static files (e.g. a website) behind a [Traefik](https://traefik.io/) reverse-proxy, which takes care of SSL certificates. It can also be configured to do anything else Nginx is capable of.

## Adjusting the playbook configuration

To enable Nginx with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# nginx                                                                #
#                                                                      #
########################################################################

nginx_enabled: true

nginx_hostname: nginx.example.com

########################################################################
#                                                                      #
# /nginx                                                               #
#                                                                      #
########################################################################
```

### Set the path prefix (optional)

By default, Nginx is served at the root of the hostname (`/`). To serve it under a subpath instead, add the following configuration to your `vars.yml` file:

```yaml
# The path prefix must either be `/` or not end with a slash (e.g. `/nginx`).
nginx_path_prefix: /nginx
```

### Serving static files

By default, Nginx serves the files found in the data directory (`/nginx/data` on the server, or wherever `nginx_data_path` points to). Put your website's files (e.g. `index.html`) there.

The directory is mounted read-only into the container, so it needs to be readable by the user Nginx runs as (see `nginx_uid` and `nginx_gid`).

### Changing what Nginx serves (optional)

What the default server does is controlled by `nginx_server_configuration`, which contains directives for the `server` context.

For example, to make Nginx reverse-proxy to another container instead, add the following configuration to your `vars.yml` file:

```yaml
nginx_server_configuration: |
  location / {
    set $backend http://my-service:8080;
    proxy_pass $backend;
  }

# The other container needs to be reachable over one of the container networks Nginx is connected to.
nginx_container_additional_networks_custom:
  - my-service-network
```

Proxying to a variable (as done above) makes Nginx resolve the container's hostname at request time. Otherwise, Nginx resolves it once at startup and refuses to start if the other container is not running.

To add configuration to the `http` context (e.g. `upstream` or `map` blocks, or additional `server` blocks), use `nginx_configuration_extension`. To take full control, you can also completely redefine `nginx_configuration`.

### Adjusting the Nginx configuration

There are some additional things you may wish to configure about the component.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for the `nginx_config_*` variables that you can customize via your `vars.yml` file.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Nginx becomes available at the specified hostname like `https://nginx.example.com`.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu nginx` (or how you/your playbook named the service, e.g. `mash-nginx`).

### Test the configuration

To check the configuration for errors, run `docker exec nginx nginx -c /config/nginx.conf -t` on the server (adjust the container name if you/your playbook named it differently, e.g. `mash-nginx`).
