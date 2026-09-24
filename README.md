<!--
SPDX-FileCopyrightText: 2022 - 2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Nginx reverse-proxy Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [Nginx](https://nginx.org/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on the [`com.devture.ansible.role.systemd_docker_base` role](https://github.com/devture/com.devture.ansible.role.systemd_docker_base).

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

## Features

- **automatic SSL certificates**: certificates are obtained from [Let's Encrypt](https://letsencrypt.org/) (or another ACME server) via Nginx's own [ACME module](https://nginx.org/en/docs/http/ngx_http_acme_module.html) — see `nginx_config_acme_issuer_enabled`

- **entrypoints**: `web` (HTTP, port 80) and `web-secure` (HTTPS, port 443, with HTTP/2 and HTTP/3 support) entrypoints, with `web` to `web-secure` redirection. Additional entrypoints can be defined via `nginx_additional_entrypoints`

- **sites**: other roles and playbooks can inject their own sites (`server` blocks) via `nginx_sites_auto` / `nginx_sites_custom`

- **status page and metrics**: the [`stub_status`](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) page can be exposed on a hostname protected by HTTP Basic authentication (`nginx_status_enabled`) and/or on a dedicated metrics port for [nginx-prometheus-exporter](https://github.com/nginx/nginx-prometheus-exporter) (`nginx_config_metrics_stub_status_enabled`)

- **health checking**: the systemd service only becomes active once Nginx responds on its internal health entrypoint, so services which depend on it start after it's ready

- **hardened container**: the container runs as a non-`root` user, with all [capabilities dropped](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) and a read-only filesystem. Unlike other reverse-proxies, Nginx does not need access to the Docker API

- **graceful reloading**: `systemctl reload nginx` applies configuration changes without restarting the container

## Usage

Example playbook:

```yaml
- hosts: servers
  roles:
    - role: galaxy/com.devture.ansible.role.systemd_docker_base

    - role: galaxy/nginx

    - role: another_role
```

Example playbook configuration (`group_vars/servers` or other):

```yaml
nginx_container_network: "{{ my_container_network }}"

nginx_uid: "{{ my_uid }}"
nginx_gid: "{{ my_gid }}"
```

## Adding sites

Nginx does not discover services on its own (like Traefik does with container labels), so each service needs a site (a `server` block) defined for it.

To make this easier, the role generates a configuration snippet for each entrypoint (at `/config/entrypoints/ENTRYPOINT_NAME.conf` in the container) containing the necessary `listen` directives, etc., and one for the ACME certificate issuer (at `/config/acme-issuers/ACME_ISSUER_NAME.conf` in the container) containing the necessary certificate directives.

Here's some example configuration (e.g. `group_vars/servers`) which defines a site for a service running in a container named `my-service` (connected to the `nginx_container_network` network):

```yaml
nginx_sites_custom:
  - |
    server {
      server_name example.com;

      include /config/entrypoints/{{ nginx_entrypoint_primary }}.conf;
      include /config/acme-issuers/{{ nginx_acme_issuer_primary }}.conf;

      location / {
        set $backend http://my-service:8080;
        proxy_pass $backend;
      }
    }
```

Proxying to a variable (as done above) makes Nginx resolve the container's hostname at request time (using Docker's embedded DNS server, see `nginx_config_http_resolver`). Otherwise, Nginx would resolve it once at startup and refuse to start at all if the container is not running.

The default proxy settings (HTTP/1.1, WebSocket support, `X-Forwarded-*` headers, etc.) are inherited from the `http` context (see `nginx_config_http_proxy_defaults_enabled`). Note that Nginx does not inherit `proxy_set_header` and `add_header` directives in a `server` or `location` block which defines any directives of the same type on its own.

Playbooks are meant to inject their sites into `nginx_sites_auto`.

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.
