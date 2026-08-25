<!--
SPDX-FileCopyrightText: 2023, 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2025 spatterIight
SPDX-FileCopyrightText: 2025, 2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Radarr Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [Radarr](https://radarr.video/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Limitations

> [!WARNING]
> A freshly installed Radarr has no authentication of its own (`AuthenticationMethod` is `None` in the `config.xml` it writes for itself), and this role does not add any. Radarr also serves its API key to unauthenticated callers on `/initialize.json`, and that key is enough to drive the whole API. Turn authentication on under *Settings -> General -> Security* in Radarr itself, or put a middleware in front of it through `radarr_container_labels_additional_labels`, before making an installation reachable from the internet.

The API key lives in `config.xml` under `radarr_data_path` (`/radarr/data/config.xml` by default). Radarr creates that file itself, with mode `0644`; the directory around it is created by this role with mode `0750` and owned by `radarr_uid`:`radarr_gid`, so the key is not readable by other users on the host.

This role configures Radarr with security in mind by doing the following:

1. Running the container as a non-root user
2. Making the filesystem read-only
3. Dropping most capabilities

Unfortunately, due to upstream requirements, some admissions had to be made:

1. Several capabilities related to permissions are added to the container
   - SETUID
   - SETGID
   - CHOWN
   - FOWNER
   - DAC_OVERRIDE
2. A `tmpfs` volume is mounted with `exec` permissions

You can read more about these upstream requirements in the documentation:

1. <https://docs.linuxserver.io/misc/non-root/>
2. <https://docs.linuxserver.io/misc/read-only/>

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

### Releases

Release tags are cut automatically. On every push to `develop` and to `main`, [`.github/workflows/autotag.yml`](./.github/workflows/autotag.yml) runs [`bin/compute-next-tag.sh`](./bin/compute-next-tag.sh), which derives the tag from `radarr_version` in [`defaults/main.yml`](./defaults/main.yml) and the tags that already exist:

- a Radarr version that has never been released starts a new counter (`v6.4.2-0`)
- otherwise the counter is incremented (`v6.3.0-1`), but only if something under `defaults/`, `meta/`, `tasks/` or `templates/` changed since the previous release — a README or CI-only change releases nothing

Because the tag is derived from the repository's state rather than from commit messages, it does not matter in which order pull requests are merged, and no human has to tag anything. [`bin/test-compute-next-tag.sh`](./bin/test-compute-next-tag.sh) exercises the computation against throwaway repositories and runs as a prek hook whenever the script or `defaults/main.yml` changes.
