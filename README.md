# ansible-role-authelia
[![CI](https://github.com/ryclarke/ansible-role-authelia/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/ryclarke/ansible-role-authelia/actions/workflows/ci.yaml) [![Ansible Galaxy Import](https://github.com/ryclarke/ansible-role-authelia/actions/workflows/publish.yaml/badge.svg)](https://github.com/ryclarke/ansible-role-authelia/actions/workflows/publish.yaml)

An opinionated Ansible role that deploys a lean, hardened [Authelia](https://www.authelia.com) SSO stack via Docker Compose: **Authelia + lldap** ([Lightweight LDAP](https://github.com/lldap/lldap)), with optional **PostgreSQL + Redis** services.

Designed for single-instance homelab or small-business deployments. Boring, proven, minimal moving parts — every container runs as a pinned unprivileged user with `cap_drop: ALL`, `no-new-privileges`, and a read-only rootfs where possible.

## Requirements

- Ansible 2.15+
- Linux-based target with Python 3
- Docker (install it yourself, or pair this role with a `docker` role — [community.docker](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/index.html) collection required for compose)

## Quick start

```yaml
# requirements.yaml
collections:
  - name: community.docker
    version: ">=3.10.0"

roles:
  - name: ryclarke.authelia
```

```bash
ansible-galaxy install -r requirements.yaml
```

Minimal playbook:

```yaml
- hosts: auth
  become: true
  roles:
    - ryclarke.authelia
```

Core variables:

```yaml
authelia_domain: "example.com"
authelia_smtp_address: "submission://smtp.example.com:587"
authelia_smtp_username: "admin@example.com"

# Secrets — set these in an ansible-vault file, NEVER in plaintext!
authelia_jwt_secret: "..."
authelia_session_secret: "..."
authelia_storage_encryption_key: "..."
authelia_smtp_password: "..."
authelia_ldap_password: "..."
authelia_ldap_jwt_secret: "..." # managed lldap only
authelia_ldap_key_seed: "..."   # managed lldap only
```
> 🔐 **Bootstrapping Secrets**: See the official guide for [Generating Secure Values](https://www.authelia.com/reference/guides/generating-secure-values/) to generate strong secret keys and hash client credentials

Deploy with `ansible-playbook --ask-vault-pass`, then open `https://auth.example.com` and create your first user in lldap.

## Configuration Philosophy & Variable Passthrough
Structured role variables match Authelia’s expected configuration structure exactly. When passed, these variables are injected directly into Authelia’s rendered template:

- **Usage Configuration**: Extends features as your application suite evolves (e.g., adding ACL rules or OIDC clients).

- **Integration Configuration**: Overriding complex dictionary variables (such as authelia_smtp, authelia_ldap, authelia_storage_postgres, authelia_redis, or authelia_authz_endpoints) replaces the block entirely without deep-merging. Ensure complete block definitions are provided when overriding defaults.

## Usage Configuration

### Access Control Rules (ACLs)
Defines access policies for protected domains. The contents of `authelia_acl_rules` map directly to Authelia's `access_control.rules` list.

- **Docs**: [Authelia Access Control](https://www.authelia.com/configuration/security/access-control/) | [Rule Operators Guide](https://www.authelia.com/reference/guides/rule-operators/)

```yaml
authelia_acl_default_policy: deny
authelia_acl_rules:
  - policy: one_factor
    domain: ["app1.example.com"]
    subject: ["group:users"]
```

### OpenID Connect (OIDC) Clients
Configures OAuth2/OIDC clients. Injected directly into Authelia's `identity_providers.oidc.clients` list.

- **Docs**: [Authelia OIDC Client Docs](https://www.authelia.com/configuration/identity-providers/openid-connect/clients/)

> ⚠️ Note: `client_secret` must be a pre-hashed digest here, not plaintext!

### Network Definitions
Defines named CIDRs for IP-based allowlists in ACL rules. Injected into `definitions.network`.

- **Docs**: [Authelia Network Definitions Docs](https://www.authelia.com/configuration/definitions/network/)

### User Attributes
Maps custom claims/attributes from identity providers. Injected into `definitions.user_attributes`.

- **Docs**: [Authelia User Attributes Docs](https://www.authelia.com/configuration/definitions/user-attributes/)

### Authorization endpoints
Customizes proxy authentication strategies.

- **Docs**: [Authelia Authz Endpoints Docs](https://www.authelia.com/configuration/miscellaneous/server-endpoints-authz/) | [Proxy Authorization Guide](https://www.authelia.com/reference/guides/proxy-authorization/)

> ⚠️ Warning: Setting authelia_authz_endpoints completely replaces Authelia's built-in default endpoints. Include all required default endpoints alongside custom ones.

### Custom Configuration Template
For radical customizations, supply a custom Jinja2 template path:
```yaml
authelia_configuration_template: /path/to/custom-configuration.yaml.j2
```

All `authelia_*` variables remain in scope within your custom template.

## Integration Configuration

Integrations control backend infrastructure connections. The role handles container deployment automatically for managed options.

### SMTP Configuration

- **Docs**: [Authelia SMTP Docs](https://www.authelia.com/configuration/notifications/smtp/)
- **Simple Setup**: Set `authelia_smtp_address`, `authelia_smtp_username`, and `authelia_smtp_password`.
- **Advanced Setup**: Define the `authelia_smtp` dictionary (replaces defaults).
- **Disabled**: Set `authelia_smtp_enabled: false` (testing only).

### LDAP Authentication

- **Docs**: [Authelia LDAP Docs](https://www.authelia.com/configuration/first-factor/ldap/) | [LDAP Integrations Guide](https://www.authelia.com/reference/integrations/ldap-integrations/)
- **Managed LLDAP (Default)**: Set `authelia_ldap_managed: true` and supply `authelia_ldap_password`.
- **External LDAP**: Set `authelia_ldap_managed: false` and supply the `authelia_ldap` dictionary.

### Persistent Storage

- **Docs**: [Authelia Storage Docs](https://www.authelia.com/configuration/storage/postgres/)
- **Local (Default)**: `authelia_storage_backend: local` (SQLite).
- **Managed Postgres**: Set `authelia_storage_backend: postgres` and `authelia_storage_postgres_managed: true`.
- **External Postgres**: Set `authelia_storage_backend: postgres`, `authelia_storage_postgres_managed: false`, and supply `authelia_storage_postgres` details.

### Redis Sessions

- **Docs**: [Authelia Redis Docs](https://www.authelia.com/configuration/session/redis/)
- **Disabled (Default)**: `authelia_redis_enabled: false`.
- **Managed / External**: Toggle `authelia_redis_enabled` and `authelia_redis_managed` appropriately.

## Extending Compose & Docker Images

### Additional Services & Environment Variables
Add sidecars or custom environment flags directly to the generated Docker Compose deployment:
```yaml
authelia_extra_compose_services:
  whoami-exporter:
    image: traefik/whoami:latest
    networks: ["{{ authelia_network_name }}"]

authelia_extra_compose_envs:
  CUSTOM_KEY: "value"
```

### Container Images & Versions

Container image definitions separate the target image registry/repository from the tag version. You can pin image tags directly using the `*_version variables`, or supply a fully qualified `*_image` override (e.g., to reference a custom registry or pin sha256 digests in production):

| Service     | Version Variable            | Default Version          | Image Variable (Default)                             |
|-------------|-----------------------------|--------------------------|------------------------------------------------------|
| Authelia    | `authelia_version`          | `latest`                 | `docker.io/authelia/authelia:{{ authelia_version }}` |
| LLDAP       | `authelia_lldap_version`    | `latest-alpine-rootless` | `docker.io/lldap/lldap:{{ authelia_lldap_version }}` |
| PostgreSQL  | `authelia_postgres_version` | `18-alpine`              | `postgres:{{ authelia_postgres_version }}`           |
| Redis       | `authelia_redis_version`    | `8-alpine`               | `redis:{{ authelia_redis_version }}`                 |

Production deployments should pin these to digests (`image@sha256:...`) — `latest` defaults are convenient for first runs but make the deployed version dependent on pull timing, which undermines reproducibility and supply-chain auditing.

## Key Variables Reference

| Variable                       | Default         | Purpose                         |
|--------------------------------|-----------------|---------------------------------|
| `authelia_root`                | `/opt/authelia` | Deployment root path            |
| `authelia_log_level`           | `info`          | Authelia log level              |
| `authelia_theme`               | `""`            | `light`, `dark`, `grey`, `auto` |
| `authelia_acl_default_policy`  | `deny`          | Default ACL policy              |
| `authelia_acl_rules`           | `[]`            | Access-control rules            |
| `authelia_oidc_clients`        | `[]`            | OpenID Connect clients          |
| `authelia_network_definitions` | `{}`            | Named CIDR definitions          |
| `authelia_user_attributes`     | `{}`            | Custom attribute mappings       |
| `authelia_authz_endpoints`     | `{}`            | Custom endpoint configurations  |

> ℹ️ Secrets are rendered into `./secrets/` and mounted securely via Docker Compose secrets.

## Contributing & Testing

Bug reports and PRs welcome!

### Local Render Harness

Run tests locally to validate syntax, template rendering, and compose output:

```make
# Run ansible-lint
make lint

# Run render tests against default fixture
make test

# Validate generated compose output with Docker/Podman
make validate
```
