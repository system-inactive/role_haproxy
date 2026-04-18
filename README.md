# haproxy

Ansible role for installing and configuring **HAProxy** on RHEL/CentOS-based systems.

Configures TCP/SNI-based frontend routing, flexible backends with optional backup servers, stats page, firewalld rules, SELinux boolean, and optional Cloudflare DNS record creation.

## Requirements

- RHEL / CentOS 7+
- `firewalld` service (installed automatically by the role)
- For DNS record creation: a Cloudflare account with API access

## Role Variables

### `defaults` — HAProxy global defaults section

```yaml
defaults:
  mode: tcp
  retries: 3
  maxconn: 3000
  timeout:
    queue: 1m
    connect: 10s
    client: 1m
    server: 1m
    check: 10s
```

### `frontend` — list of frontend listeners

Each frontend uses SNI-based routing (`tcp-request content accept if { req_ssl_hello_type 1 }`).

```yaml
frontend:
  - name: example.com
    port: 443
    backend: example_backend
  - name: api.example.com
    port: 443
    backend: api_backend
```

| Field | Description |
|---|---|
| `name` | Frontend name, also used as SNI hostname for routing |
| `port` | Port to bind on `0.0.0.0` |
| `backend` | Name of the backend to route matching traffic to |

### `backend` — list of backend server pools

```yaml
backend:
  - name: example_backend
    balance: roundrobin
    servers:
      - name: web01
        hostname: 192.168.1.10
        port: 443
      - name: web02
        hostname: 192.168.1.11
        port: 443
        backup: true
  - name: api_backend
    balance: leastconn
    servers:
      - name: api01
        hostname: 192.168.1.20
        port: 8443
```

| Field | Description |
|---|---|
| `name` | Backend name (referenced in frontend) |
| `balance` | Load balancing algorithm: `roundrobin`, `leastconn`, `source`, etc. |
| `servers` | List of backend servers |
| `servers[].name` | Server label in HAProxy config |
| `servers[].hostname` | IP address or hostname |
| `servers[].port` | Backend port |
| `servers[].backup` | Optional. Set to `true` to mark server as backup |

### `listen` — stats page

```yaml
listen:
  port: 8080
  user: admin
  password: strongpassword
```

Stats page will be available at `http://<host>:8080/`.

### `dns` — optional Cloudflare DNS records

If defined, the role creates DNS A records in Cloudflare after deploying HAProxy.

```yaml
dns:
  - domain: example.com
    record: haproxy
    value: 203.0.113.10
    email: admin@example.com
    token: your_cloudflare_api_key
```

## Example Playbook

```yaml
- hosts: haproxy_servers
  roles:
    - role: haproxy
      vars:
        defaults:
          mode: tcp
          retries: 3
          maxconn: 3000
          timeout:
            queue: 1m
            connect: 10s
            client: 1m
            server: 1m
            check: 10s

        frontend:
          - name: example.com
            port: 443
            backend: example_backend

        backend:
          - name: example_backend
            balance: roundrobin
            servers:
              - name: web01
                hostname: 192.168.1.10
                port: 443
              - name: web02
                hostname: 192.168.1.11
                port: 443
                backup: true

        listen:
          port: 8080
          user: admin
          password: strongpassword
```

## What the role does

1. Updates system packages and installs HAProxy
2. Deploys `/etc/haproxy/haproxy.cfg` from template (with config validation before apply)
3. Installs and configures firewalld — opens frontend ports and stats port
4. Configures SELinux boolean `haproxy_connect_any` (when SELinux is enabled)
5. Optionally creates Cloudflare DNS A records
6. Restarts HAProxy and firewalld via handlers

## Notes

- HAProxy config is validated with `haproxy -c -f` before being applied — a broken config will not be deployed
- The role targets **RHEL/CentOS** (uses `yum`). For Debian/Ubuntu-based systems the package manager tasks need adjustment
- Stats page credentials are stored in plaintext in the config — use Ansible Vault for `listen.password` in production

## License

MIT
