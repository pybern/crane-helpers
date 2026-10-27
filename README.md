# crane-helpers

A guide to using [`crane`](https://github.com/google/go-containerregistry/tree/main/cmd/crane) from the CLI to pull images from Docker Hub and push them to a private Harbor registry that uses username and password credentials.

`crane` talks straight to registries, so it does **not** need a Docker daemon. It is a good fit for CI runners, air-gapped mirroring, and bastion hosts.

---

## Contents

1. [Install crane](#1-install-crane)
2. [Prepare Harbor](#2-prepare-harbor)
3. [Log in](#3-log-in)
4. [Option A: copy directly from Docker Hub to Harbor (recommended)](#4-option-a-copy-directly-from-docker-hub-to-harbor-recommended)
5. [Option B: pull to a tarball, then push](#5-option-b-pull-to-a-tarball-then-push-air-gapped--two-step)
6. [Verify the upload](#6-verify-the-upload)
7. [Mirror a list of images](#7-mirror-a-list-of-images)
8. [TLS, self-signed certificates and HTTP registries](#8-tls-self-signed-certificates-and-http-registries)
9. [Troubleshooting](#9-troubleshooting)
10. [Command cheat sheet](#10-command-cheat-sheet)

The examples use these values. Replace them with your own:

| Placeholder           | Meaning                               |
|-----------------------|---------------------------------------|
| `harbor.example.local` | Harbor hostname (and port, if not 443) |
| `library`             | Harbor project to push into           |
| `myuser` / `mypassword` | Harbor credentials                    |

---

## 1. Install crane

**Prebuilt binary (Linux/macOS):**

```bash
VERSION=$(curl -s "https://api.github.com/repos/google/go-containerregistry/releases/latest" | jq -r '.tag_name')
OS=Linux        # or Darwin
ARCH=x86_64     # or arm64
curl -sL "https://github.com/google/go-containerregistry/releases/download/${VERSION}/go-containerregistry_${OS}_${ARCH}.tar.gz" \
  | tar -xzf - crane
sudo mv crane /usr/local/bin/
```

**Homebrew:**

```bash
brew install crane
```

**Go toolchain:**

```bash
go install github.com/google/go-containerregistry/cmd/crane@latest
```

Check that it works:

```bash
crane version
```

---

## 2. Prepare Harbor

Harbor only accepts pushes into a **project** that already exists. Before you push:

1. In the Harbor UI, open **Projects → New Project** and create one (for example `library`, which usually exists by default).
2. Make sure your user is a **Developer**, **Maintainer** or **Project Admin** on that project. The Guest and Limited Guest roles cannot push.
3. (Optional, for CI) Create a **Robot Account** with push permission and use its name and secret as the username and password below.

The image path in Harbor is always `<harbor-host>/<project>/<repository>:<tag>`.

---

## 3. Log in

### Harbor (required)

```bash
crane auth login harbor.example.local -u myuser -p 'mypassword'
```

To keep the password out of your shell history and process list, read it from stdin:

```bash
echo "$HARBOR_PASSWORD" | crane auth login harbor.example.local -u myuser --password-stdin
```

> **Robot accounts:** Harbor robot account names contain a `$`, for example `robot$ci-pusher`. Put them in **single quotes** so the shell does not expand the `$`:
>
> ```bash
> echo "$ROBOT_SECRET" | crane auth login harbor.example.local -u 'robot$ci-pusher' --password-stdin
> ```

### Docker Hub (optional, but recommended)

Pulling public images works without logging in. Anonymous pulls are rate-limited, though, so log in if you mirror many images:

```bash
echo "$DOCKERHUB_TOKEN" | crane auth login index.docker.io -u mydockerhubuser --password-stdin
```

### Where credentials are stored

`crane auth login` writes to the standard Docker config file, `~/.docker/config.json`. It also reads any credentials already there, such as those from `docker login`, and any configured credential helpers.

To keep credentials away from your home directory (useful in CI), point `DOCKER_CONFIG` at a temporary directory:

```bash
export DOCKER_CONFIG=$(mktemp -d)
echo "$HARBOR_PASSWORD" | crane auth login harbor.example.local -u myuser --password-stdin
# ... crane commands ...
rm -rf "$DOCKER_CONFIG"
```

To log out:

```bash
crane auth logout harbor.example.local
```

---

## 4. Option A: copy directly from Docker Hub to Harbor (recommended)

`crane copy` (alias `crane cp`) streams the image from registry to registry and does not write anything to disk. It keeps **all architectures** of a multi-arch image and preserves the digest.

```bash
crane copy docker.io/library/nginx:1.27 harbor.example.local/library/nginx:1.27
```

Other forms that are often useful:

```bash
# Official images can use the short name: "nginx" == "docker.io/library/nginx"
crane copy nginx:1.27 harbor.example.local/library/nginx:1.27

# Non-official images include the Docker Hub namespace
crane copy docker.io/bitnami/redis:7.4 harbor.example.local/library/redis:7.4

# Copy only one platform (this produces a single-arch image in Harbor)
crane copy --platform linux/amd64 nginx:1.27 harbor.example.local/library/nginx:1.27-amd64

# Copy every tag of a repository (can be very large, so use with care)
crane copy --all-tags docker.io/library/alpine harbor.example.local/library/alpine

# Pin to an immutable digest
crane copy nginx@sha256:<digest> harbor.example.local/library/nginx:1.27
```

---

## 5. Option B: pull to a tarball, then push (air-gapped / two-step)

Use this option when the machine that can reach Docker Hub cannot reach Harbor, or when you want an offline copy of the image.

### Step 1: pull (on a machine with internet access)

```bash
# Pulls a single platform (crane picks one by default; set it explicitly to be safe)
crane pull --platform linux/amd64 nginx:1.27 nginx_1.27_amd64.tar
```

To save **several images** into one tarball:

```bash
crane pull --platform linux/amd64 nginx:1.27 redis:7.4 images.tar
```

To keep a **multi-arch** image intact, save it as an OCI image layout directory instead of a tarball:

```bash
crane pull --format=oci nginx:1.27 ./nginx-oci
tar -czf nginx-oci.tar.gz nginx-oci     # for transfer
```

### Step 2: move the file

Copy the `.tar` or `.tar.gz` file to the machine that can reach Harbor, using `scp`, USB storage, an artifact store or similar.

### Step 3: push (on a machine that can reach Harbor)

Log in to Harbor first (see [section 3](#3-log-in)), then push:

```bash
# Single-image tarball
crane push nginx_1.27_amd64.tar harbor.example.local/library/nginx:1.27

# Multi-arch OCI layout: push it as an index so every platform is kept
tar -xzf nginx-oci.tar.gz
crane push --index ./nginx-oci harbor.example.local/library/nginx:1.27
```

> A plain `crane push` expects one image per tarball. If a path contains several images, `crane push` needs `--index`, which combines them into **one** multi-image index under a single tag. To push images under separate names, pull each one into its own tarball.

---

## 6. Verify the upload

```bash
# List tags in the Harbor repository
crane ls harbor.example.local/library/nginx

# Compare digests between source and destination (they match for crane copy)
crane digest nginx:1.27
crane digest harbor.example.local/library/nginx:1.27

# Inspect the manifest (shows the platform list for a multi-arch image)
crane manifest harbor.example.local/library/nginx:1.27 | jq .

# Inspect the image config (env, entrypoint, labels, ...)
crane config harbor.example.local/library/nginx:1.27 | jq .
```

You can also confirm in the Harbor UI under **Projects → library → Repositories**.

---

## 7. Mirror a list of images

Create `images.txt` with one Docker Hub image per line:

```text
nginx:1.27
redis:7.4
postgres:16
bitnami/kafka:3.7
```

Then run this script, saved as `mirror.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

HARBOR="${HARBOR:-harbor.example.local}"
PROJECT="${PROJECT:-library}"
LIST="${1:-images.txt}"

: "${HARBOR_USER:?set HARBOR_USER}"
: "${HARBOR_PASSWORD:?set HARBOR_PASSWORD}"

echo "$HARBOR_PASSWORD" | crane auth login "$HARBOR" -u "$HARBOR_USER" --password-stdin

while IFS= read -r src || [[ -n "$src" ]]; do
  [[ -z "$src" || "$src" == \#* ]] && continue
  # "bitnami/kafka:3.7" -> "kafka:3.7"; "nginx:1.27" -> "nginx:1.27"
  name_tag="${src##*/}"
  dst="${HARBOR}/${PROJECT}/${name_tag}"
  echo ">> ${src} -> ${dst}"
  crane copy "$src" "$dst"
done < "$LIST"
```

Run it:

```bash
chmod +x mirror.sh
HARBOR_USER=myuser HARBOR_PASSWORD='mypassword' ./mirror.sh images.txt
```

---

## 8. TLS, self-signed certificates and HTTP registries

Many local Harbor installations use a self-signed or internal CA certificate, or plain HTTP.

**Trust your CA (preferred).** On Linux, crane is a Go binary and reads `SSL_CERT_FILE` or `SSL_CERT_DIR`:

```bash
export SSL_CERT_FILE=/path/to/harbor-ca.crt
crane auth login harbor.example.local -u myuser -p 'mypassword'
```

You can also install the CA certificate into the OS trust store:

```bash
# Debian/Ubuntu
sudo cp harbor-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates
# RHEL/Fedora
sudo cp harbor-ca.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-certificates   # or: update-ca-trust
```

**Skip TLS verification, or use HTTP (lab use only).** Add the global `--insecure` flag to each command:

```bash
crane auth login --insecure harbor.example.local:8080 -u myuser -p 'mypassword'
crane copy --insecure nginx:1.27 harbor.example.local:8080/library/nginx:1.27
```

If Harbor listens on a port other than 443, include the port in **every** reference (login and image names). The credential entry is keyed by `host:port`.

---

## 9. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `UNAUTHORIZED: unauthorized to access repository` | You are not logged in to that exact `host[:port]`, or the password is wrong. Run `crane auth login` again with the same host string you use in the image reference. |
| `DENIED: requested access to the resource is denied` / `project not found` | The Harbor project does not exist, or your user or robot account lacks push permission on it. See [section 2](#2-prepare-harbor). |
| `x509: certificate signed by unknown authority` | Harbor uses a self-signed or internal CA. See [section 8](#8-tls-self-signed-certificates-and-http-registries). |
| `http: server gave HTTP response to HTTPS client` | Harbor is serving plain HTTP. Add `--insecure`. |
| `TOOMANYREQUESTS: You have reached your pull rate limit` | Docker Hub anonymous rate limit. Log in to `index.docker.io` (see [section 3](#3-log-in)). |
| Robot account login fails | The `$` in `robot$name` was expanded by the shell. Wrap the username in single quotes. |
| Only one architecture shows up in Harbor | You used `crane pull` to a tarball, or `--platform`. Use `crane copy` or `crane pull --format=oci` followed by `crane push <dir>`. |
| Push is slow or times out | Large layers or a slow link. Retry; crane skips layers that already exist in Harbor. |

Add `-v` (verbose) to any crane command to see the raw HTTP requests:

```bash
crane -v copy nginx:1.27 harbor.example.local/library/nginx:1.27
```

---

## 10. Command cheat sheet

```bash
# Auth
crane auth login  <harbor-host> -u <user> --password-stdin
crane auth logout <harbor-host>

# Registry to registry (all archs, no disk)
crane copy <src-image> <harbor-host>/<project>/<repo>:<tag>

# Two-step via tarball
crane pull --platform linux/amd64 <src-image> image.tar
crane push image.tar <harbor-host>/<project>/<repo>:<tag>

# Two-step, keeping multi-arch
crane pull --format=oci <src-image> ./oci-dir
crane push ./oci-dir <harbor-host>/<project>/<repo>:<tag>

# Inspect
crane ls       <harbor-host>/<project>/<repo>
crane digest   <image>
crane manifest <image>
crane config   <image>

# Retag inside Harbor (no data transfer)
crane tag <harbor-host>/<project>/<repo>:<tag> <new-tag>

# Delete a tag or manifest (requires delete permission in Harbor)
crane delete <harbor-host>/<project>/<repo>@sha256:<digest>
```

Full reference: <https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md>
