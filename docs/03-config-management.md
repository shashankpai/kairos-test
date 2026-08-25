# Config management

How runtime configuration (users, packages, files, boot stages) reaches the Pi without re-flashing.

## Mechanism

Kairos applies every `*.yaml` file in `/oem/` in lexicographic order on every boot. The flash-time cloud-config (see [01-flashing.md](01-flashing.md)) installs a `stages.boot.git` entry that clones this repo into `/oem/cloud-config-files/` on each boot. So:

```
this repo (cloud-config/*.yaml)
        |
        |  git push to main
        v
   Pi reboots
        |
        |  stages.boot.git pulls repo -> /oem/cloud-config-files/
        v
   Kairos applies /oem/*.yaml + /oem/cloud-config-files/*.yaml in lex order
```

## File naming convention

Files in `cloud-config/` are numbered to control apply order:

| File | Purpose |
| --- | --- |
| `00_base.yaml` | Users, SSH keys, k3s enable. Always present. |
| `10_git-pull.yaml` | The boot-time git pull itself. Must be in the flash-time config too (chicken-and-egg — see below). |
| `20_*.yaml` ... | Additional runtime config: extra packages via `stages.boot.commands`, write_files, services, etc. Add as needed. |

## Making a config change

1. Edit or add a file under `cloud-config/` in this repo.
2. Commit and push to `main`.
3. Reboot the Pi (or wait for its next reboot):
   ```bash
   ssh kairos@<pi> sudo reboot
   ```
4. After boot, verify:
   ```bash
   ssh kairos@<pi>
   ls /oem/cloud-config-files/   # repo contents present
   journalctl -fu kairos-agent   # no stage errors
   ```

## The chicken-and-egg

`10_git-pull.yaml` defines the git pull, but the git pull is what delivers `10_git-pull.yaml`. Resolution: the **flash-time** cloud-config (the local, gitignored `build/cloud-config.yaml`) must already contain the `stages.boot.git` entry. After the first boot, the repo's copy is re-pulled each boot and stays in sync. If you ever change the git URL, branch, or auth in `10_git-pull.yaml`, you must also update the flash-time config and re-flash — the Pi can't pull a changed pull-instruction from a repo it can't yet reach.

## Private repo auth

If this repo is private, the `stages.boot.git` entry needs a GitHub deploy key:

```yaml
stages:
  boot:
    - name: "Pull config repo"
      git:
        url: "git@github.com:shashankpai/kairos-test.git"
        path: "/oem/cloud-config-files"
        branch: "main"
        auth:
          insecure: true
          private_key: |
            -----BEGIN OPENSSH PRIVATE KEY-----
            ...
            -----END OPENSSH PRIVATE KEY-----
```

The private key lives **only** in the flash-time `build/cloud-config.yaml` (gitignored). The repo's `cloud-config/10_git-pull.yaml` omits the `auth` block (or uses a placeholder) so the secret is never committed.

## What belongs in cloud-config vs. the image

| Change type | Where |
| --- | --- |
| Add a user, SSH key, write a file, start a service, run a one-off command | `cloud-config/*.yaml` |
| Add a package to the OS itself, change the kernel, change the base distro | `image/Dockerfile` + rebuild + upgrade |

Cloud-config is applied at boot and is mutable per-reboot; the image is immutable per-version. Prefer cloud-config for anything that doesn't require a new OS image.
