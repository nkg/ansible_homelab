# nkg.homelab

Ansible collection of **Tier 1 service roles for a self-hosted homelab** —
the things that run *on* the hypervisors rather than the hypervisors
themselves. Hypervisor provisioning lives in
[`nkg.proxmox`](https://github.com/nkg/ansible_proxmox); this collection is
what you put on the VMs afterwards.

Every role targets **Debian** (bookworm/trixie) and is **vendor-neutral** —
nothing assumes a hosting provider, an overlay network, or a particular
network topology. Where a role needs to reach another host, that address is a
variable you set.

## Roles

| Role | What it does |
|---|---|
| `pbs` | Proxmox Backup Server + an initial datastore, for receiving backups from PVE nodes |
| `garage_native` | Garage S3-compatible object storage, native systemd unit — no Docker |
| `postgres_native` | PostgreSQL cluster from the PGDG apt repo, with tuning and `pg_hba` — no Docker |
| `komodo` | Komodo Core via Docker Compose, plus the Periphery agent under systemd |
| `restic` | restic snapshots on a schedule, with retention and optional database pre-dumps |
| `backups` | Local database and file backup scripts, on a cron schedule with logrotate |
| `alloy` | Grafana Alloy agent for shipping metrics and logs |

`_native` in a name means "packages and systemd, not a container". Those roles
exist because a database or an object store on a single homelab box is easier
to reason about, back up and recover without a container runtime in the path.

## Install

```bash
ansible-galaxy collection install git+https://github.com/nkg/ansible_homelab.git
ansible-galaxy install -r requirements.yml   # the standalone roles below
```

`requirements.yml` carries what `galaxy.yml` cannot express: `komodo`
delegates Periphery installation to the standalone `bpbradley.komodo` role
rather than reimplementing it, then layers its own `overrides.toml` on top via
a second `--config-path`, so upstream can rewrite its own file freely.

Collection dependencies (`community.docker`, `community.postgresql`) are
declared in `galaxy.yml` and resolved by Galaxy automatically.

## Use

```yaml
- hosts: pbs
  roles:
    - nkg.homelab.pbs

- hosts: data
  roles:
    - role: nkg.homelab.postgres_native
      vars:
        postgres_native_superuser_password: "{{ vault_pg_superuser }}"
    - role: nkg.homelab.restic
      vars:
        restic_repository: "s3:https://s3.example.net/backups"
        restic_password: "{{ vault_restic_password }}"
```

Each role's tunables are documented in its `defaults/main.yml`, which is the
authoritative list — the tables here would go stale.

## Secrets

Several roles **refuse to run** rather than come up insecure:

- `garage_native` asserts `rpc_secret`, `admin_token` and `metrics_token` are
  all non-empty.
- `komodo` asserts the DB password is not `changeme` and that at least one
  real Periphery public key is configured — blank and placeholder entries do
  not count.
- `postgres_native` asserts a superuser password is set.

Supply these from a vault or a secrets manager. None of them have usable
defaults, deliberately: a default secret is worse than a missing one, because
it works.

## Testing

`restic`, `backups` and `alloy` carry molecule scenarios:

```bash
molecule test -s default   # from within the role directory
```

Molecule scenarios are excluded from the built artifact via `build_ignore`.

CI runs yamllint and ansible-lint at the **production** profile on every push
and pull request.

## Provenance

These roles ran in production before this collection existed, as a loose
directory of role folders with no `galaxy.yml`, no metadata and no tests
wired up. Extracting them added the collection scaffolding, `meta/main.yml`
for each role, and a lint gate — and fixed one latent bug in the process:
`restic`'s molecule verify registered `restic_version`, shadowing the role
default of the same name for the rest of the play. ansible-lint does not
catch that, because the name already carries the required role prefix, so it
passes `var-naming` while being exactly the collision the rule exists to
prevent. See `changelogs/changelog.yaml`.

Roles deliberately **not** brought across: `ansible-runners` and `runners`
(owned by the runner platform's own repo), `hetzner_storage` (provider
specific), and the `docker`, `common` and `crowdsec` roles (which overlap
with `nkg.proxmox` and want reconciling before they move, not copying).

## Licence

GPL-2.0-or-later. See [LICENSE](LICENSE).
