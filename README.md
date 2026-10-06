# gitlab_runner

Ansible role that installs GitLab Runner on Ubuntu from the official apt repository and manages `/etc/gitlab-runner/config.toml` from a template.

The role is built around a few rules:

- packages come from GitLab's own apt repository, added as a deb822 source with an explicit `signed_by` key, not from a downloaded binary
- `config.toml` is generated from role variables, so the file on the server always matches the inventory
- the runner version is pinned, and `gitlab-runner` and `gitlab-runner-helper-images` are installed together at that one version
- the token never reaches the log by default

## What it does

In the order the tasks run:

1. **Installs `python3-debian`.** The `ansible.builtin.deb822_repository` module needs it on the target. The apt cache is refreshed first if it is older than an hour, so the install also works on a fresh host with empty package lists. The task runs with `check_mode: false`, so that a `--check` run still has the library present and the repository task can be evaluated.

2. **Adds the GitLab Runner apt repository** as a deb822 source named `gitlab-runner`. The URI comes from `gitlab_runner_repository_url`, the suite from `ansible_facts.distribution_release`, the component is `main`, and the signing key is fetched from `gitlab_runner_repository_gpg_key_url` via `signed_by`. Changes notify an apt cache update.

3. **Flushes handlers** with `ansible.builtin.meta: flush_handlers`, so the apt cache is refreshed before anything is installed from the new repository rather than at the end of the play.

4. **Installs `gitlab-runner` and `gitlab-runner-helper-images`**, both pinned to `gitlab_runner_version`, in a single apt transaction.

5. **Enables and starts the `gitlab-runner` service.**

6. **Writes `/etc/gitlab-runner/config.toml`** from `templates/config.toml.j2`, owner `root`, group `root`, mode `0600`. The task honours `gitlab_runner_no_log`.

7. **Adds the `gitlab-runner` user to `gitlab_runner_groups`** with `append: true`, only when the list is not empty. A change here notifies a restart of the service.

The role does not register runners, does not install Docker, and does not create the groups it adds the user to.

## Requirements

- **ansible-core 2.15 or newer** on the control machine. The role uses `ansible.builtin.deb822_repository`, which was added in 2.15.
- **Ubuntu 24.04 (noble)** on the target. That is the only platform the role has been checked on.
- **Fact gathering enabled.** The repository URI and suite are built from `ansible_facts['distribution']` and `ansible_facts.distribution_release`, so a play with `gather_facts: false` cannot add the repository.
- **`become: true`.** The role installs packages, writes under `/etc` and manages a systemd unit.
- **`python3-debian` is installed by the role itself.** You do not need to pre-install it.
- **The groups in `gitlab_runner_groups` must already exist.** The role appends the `gitlab-runner` user to them and does not create them. A missing group fails the task.
- **Docker, if you need it, must be installed before this role runs.** This applies both to the `docker` executor and to the common `shell` executor setup where jobs call the `docker` CLI themselves — in that case Docker must be present and the `docker` group must exist, because `gitlab_runner_groups` is how the runner user gets access to the socket.

## Role variables

All variables use the `gitlab_runner_` prefix. Defaults are in `defaults/main.yml`.

### Packages and repository

| Variable | Type | Default | Description |
|---|---|---|---|
| `gitlab_runner_version` | string | `"19.1.0*"` | apt version specification, applied to both `gitlab-runner` and `gitlab-runner-helper-images` |
| `gitlab_runner_repository_url` | string | `https://packages.gitlab.com/runner/gitlab-runner/{{ ansible_facts['distribution'] \| lower }}` | Repository URI |
| `gitlab_runner_repository_gpg_key_url` | string | `https://packages.gitlab.com/gpgkey/gpg.key` | Key used for `signed_by` |

### Global `config.toml` settings

These three are written to the top of `config.toml` exactly as given.

| Variable | Type | Default | Description |
|---|---|---|---|
| `gitlab_runner_concurrent` | int | `1` | `concurrent` — total jobs run in parallel across all runners |
| `gitlab_runner_check_interval` | int | `0` | `check_interval` — seconds between job checks; `0` leaves GitLab Runner on its built-in default |
| `gitlab_runner_shutdown_timeout` | int | `0` | `shutdown_timeout` — graceful shutdown limit; `0` leaves GitLab Runner on its built-in default |

### Role behaviour

| Variable | Type | Default | Description |
|---|---|---|---|
| `gitlab_runner_no_log` | bool | `true` | Hides the output of the `config.toml` task, so the token is not printed |
| `gitlab_runner_groups` | list of strings | `[]` | Extra groups for the `gitlab-runner` user, appended. Skipped when empty |
| `gitlab_runner_runners` | list of dicts | `[]` | Runner sections to write. Empty means a `config.toml` with global settings and no `[[runners]]` block |

### Structure of a `gitlab_runner_runners` item

| Key | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | `name` — the runner's description in `config.toml` |
| `url` | string | no, defaults to `https://gitlab.com` | `url` — GitLab instance the runner talks to |
| `token` | string | yes | `token` — runner authentication token, see Important notes |
| `executor` | string | yes | `executor` — for example `docker` or `shell` |
| `docker` | dict | no | Rendered as a `[runners.docker]` table, see below |

`name`, `token` and `executor` have no default in the template. Leaving one out fails the run with an undefined-variable error.

**Only these five keys are rendered.** Any other key you put on the item is ignored silently. In particular `tags`, `limit`, `environment`, `builds_dir`, `output_limit`, `request_concurrency` and `[runners.cache]` are not supported by the template.

### The `docker` dict

The keys of `docker` are written into `[runners.docker]` as they are: the key name verbatim, the value through the `to_json` filter. That makes the mapping from YAML types to TOML straightforward — strings quote themselves, integers and booleans come out as `1` and `true`, and lists come out as TOML arrays:

```yaml
docker:
  image: "alpine:3.20"
  privileged: false
  shm_size: 300000
  volumes:
    - "/cache"
```

becomes

```toml
  [runners.docker]
    image = "alpine:3.20"
    privileged = false
    shm_size = 300000
    volumes = ["/cache"]
```

Because every key becomes one `key = value` line, **only flat Docker parameters work.** Anything that is a nested table in `config.toml` cannot be expressed here: `[runners.docker.services]`, `[[runners.docker.services]]`, `[runners.docker.tmpfs]` and the like. Passing a dict as a value produces a JSON object on the right-hand side, which is not valid TOML in this position.

There is no validation of key names. A typo is written to `config.toml` as given and surfaces when the runner reads the file, not when the role runs.

## Quick start

This repository is an Ansible role. A role has no inventory and no playbook of its own, so it cannot be run directly — you call it from a small project of your own.

1. Install the role:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

   with `requirements.yml`:

   ```yaml
   roles:
     - name: gitlab_runner
       src: https://github.com/ituch136/ansible_role_gitlab_runner.git
       scm: git
       version: v1.0.0
   ```

2. Create the runner in the GitLab UI and copy the authentication token it shows you. See Important notes.

3. Put the token in a vault file:

   ```bash
   ansible-vault create group_vars/runners/vault.yml
   ```

   ```yaml
   vault_gitlab_runner_token: "glrt-REPLACE_WITH_YOUR_TOKEN"
   ```

4. Write the playbook and run it.

### Example: docker executor

```yaml
- name: Set up GitLab Runner
  hosts: runners
  become: true
  roles:
    - gitlab_runner
  vars:
    gitlab_runner_concurrent: 4
    gitlab_runner_runners:
      - name: "docker-runner"
        url: "https://gitlab.com"
        token: "{{ vault_gitlab_runner_token }}"
        executor: "docker"
        docker:
          image: "alpine:3.20"
          privileged: false
          volumes:
            - "/cache"
```

### Example: shell executor with Docker access

Here jobs run as the `gitlab-runner` user on the host and call the `docker` CLI themselves, so the user needs to be in the `docker` group. Docker must already be installed — this role does not install it, and the task fails if the group does not exist.

```yaml
- name: Set up GitLab Runner
  hosts: runners
  become: true
  roles:
    - gitlab_runner
  vars:
    gitlab_runner_concurrent: 2
    gitlab_runner_groups:
      - docker
    gitlab_runner_runners:
      - name: "shell-runner"
        url: "https://gitlab.com"
        token: "{{ vault_gitlab_runner_token }}"
        executor: "shell"
```

Run it:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

Adding the runner user to the `docker` group gives that user root-equivalent control of the host through the Docker socket. That is inherent to this setup, not something the role introduces, and it is a reason to keep such runners off shared projects.

## Important notes

**`config.toml` belongs to the role.** The whole file is generated from the template on every run. Anything edited by hand on the server — extra runners, a `[runners.cache]` section, a tweaked `concurrent` — is overwritten the next time the role runs, without warning. Everything you want in that file has to come from role variables.

**Tokens are runner authentication tokens (`glrt-`).** You create the runner in the GitLab UI, GitLab shows you a `glrt-` token, and you hand that token to this role. There is no separate `gitlab-runner register` step: with an authentication token the runner's identity already exists on the GitLab side, and `config.toml` only points at it. The consequence is that **tags, `run_untagged` and `locked` are properties of the runner in GitLab, set in the UI, and are not in `config.toml` at all.** Do not look for role variables for them — changing them is a UI operation.

**`gitlab_runner_no_log` is `true` by default.** The `config.toml` task is silenced, so a failure there reports very little and `--diff` shows nothing. To see what is actually being written, pass:

```bash
ansible-playbook playbook.yml --check --diff -e gitlab_runner_no_log=false
```

This prints the rendered `config.toml`, **including the token**, to your terminal and into any log or CI output capturing the run. Use it for debugging, and do not leave it set.

**The two packages are installed as one transaction at one version.** `gitlab-runner` and `gitlab-runner-helper-images` are a hard dependency pair — a version mismatch between them breaks jobs. They are listed in a single apt task on purpose, so that apt resolves them together. Do not split that task or pin the two separately.

**Groups are appended.** The `gitlab-runner` user is managed with `append: true`, so the role adds the groups in `gitlab_runner_groups` and never removes the user from anything else. Removing a group from the variable does not remove it from the user — that is a manual step on the host.

**The version is a wildcard.** `gitlab_runner_version` defaults to `"19.1.0*"`, an apt version glob that matches the distribution suffix of the package (`19.1.0-1`, and so on). The trailing `*` is required for the version to match; a bare `"19.1.0"` matches nothing and the install fails. Pin a different release by changing the number and keeping the `*`.

## Migrating from a binary installation

If the host already runs GitLab Runner from a binary dropped into `/usr/local/bin` and a service installed with `gitlab-runner install`, the apt package will not take over cleanly: the old systemd unit points at the old binary, and `/usr/local/bin` comes before `/usr/bin` on most paths, so `gitlab-runner` on the command line keeps resolving to the old file.

**The role does not do this migration, deliberately.** Tearing down a service and moving binaries is destructive, it is not idempotent, and getting it wrong on a host with running jobs is worse than doing it by hand once. These are manual steps, run on the host before the first role run:

1. Stop the service:

   ```bash
   sudo systemctl stop gitlab-runner
   ```

2. Remove the unit the binary installed:

   ```bash
   sudo gitlab-runner uninstall
   ```

3. Move the binary aside, rather than deleting it, so you can go back:

   ```bash
   sudo mv /usr/local/bin/gitlab-runner /root/gitlab-runner.bak
   ```

   Keep a copy of the existing `/etc/gitlab-runner/config.toml` too, since the role will overwrite it. Use it to fill in `gitlab_runner_runners`.

4. Run the role.

5. Check that the service now runs the packaged binary:

   ```bash
   systemctl show -p ExecStart gitlab-runner
   ```

   `ExecStart` should point at `/usr/bin/gitlab-runner`. If it still points into `/usr/local/bin`, the old unit was not removed in step 2.

6. Check that the runner can reach GitLab with the configured token:

   ```bash
   sudo gitlab-runner verify
   ```

Runners registered with an old-style registration token are a separate problem from the binary: `verify` tells you whether the token in `config.toml` still works. If it does not, create the runner in the UI and use the `glrt-` token it gives you.

## License

MIT
