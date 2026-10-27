# crane-helpers

A Windows guide to using [`crane`](https://github.com/google/go-containerregistry/tree/main/cmd/crane) from the command line. It pulls images from public Docker Hub, **through a corporate HTTP proxy if you have one**, and pushes them to a local Harbor registry that uses username and password credentials.

`crane` is a single `crane.exe` that talks straight to registries. It does **not** need Docker Desktop, WSL or admin rights.

All commands are **PowerShell** (Windows PowerShell 5.1 or PowerShell 7+) unless marked otherwise.

---

## Contents

1. [Install crane on Windows](#1-install-crane-on-windows)
2. [Configure the proxy](#2-configure-the-proxy)
3. [Prepare Harbor](#3-prepare-harbor)
4. [Log in](#4-log-in)
5. [Option A: copy directly from Docker Hub to Harbor (recommended)](#5-option-a-copy-directly-from-docker-hub-to-harbor-recommended)
6. [Option B: pull to a file, then push](#6-option-b-pull-to-a-file-then-push)
7. [Verify the upload](#7-verify-the-upload)
8. [Mirror a list of images (PowerShell script)](#8-mirror-a-list-of-images-powershell-script)
9. [Certificates: Harbor CA and TLS-inspecting proxies](#9-certificates-harbor-ca-and-tls-inspecting-proxies)
10. [Troubleshooting](#10-troubleshooting)
11. [Command cheat sheet](#11-command-cheat-sheet)

The examples use these values. Replace them with your own:

| Placeholder | Meaning |
|---|---|
| `harbor.example.local` | Harbor hostname (add `:port` if it is not 443) |
| `library` | Harbor project to push into |
| `myuser` | Harbor username |
| `http://proxy.corp.local:8080` | Your corporate proxy |

---

## 1. Install crane on Windows

Windows 10 (1803+) and Windows 11 include `tar.exe`, which can unpack the release archive.

```powershell
# If the download itself must go through the proxy, set it here; otherwise use $web = @{}
$web = @{ Proxy = 'http://proxy.corp.local:8080'; ProxyUseDefaultCredentials = $true }

$version = (Invoke-RestMethod 'https://api.github.com/repos/google/go-containerregistry/releases/latest' @web).tag_name
$arch    = if ($env:PROCESSOR_ARCHITECTURE -eq 'ARM64') { 'arm64' } else { 'x86_64' }
$url     = "https://github.com/google/go-containerregistry/releases/download/$version/go-containerregistry_Windows_$arch.tar.gz"
$dest    = "$env:LOCALAPPDATA\Programs\crane"

New-Item -ItemType Directory -Force -Path $dest | Out-Null
Invoke-WebRequest $url -OutFile "$env:TEMP\crane.tar.gz" -UseBasicParsing @web
tar -xzf "$env:TEMP\crane.tar.gz" -C $dest crane.exe

# Add to your user PATH (permanent) and to the current session
$userPath = [Environment]::GetEnvironmentVariable('Path', 'User')
if ($userPath -notlike "*$dest*") {
    [Environment]::SetEnvironmentVariable('Path', "$userPath;$dest", 'User')
}
$env:Path += ";$dest"

crane version
```

> `ProxyUseDefaultCredentials` sends your Windows login to the proxy, so this download also works with NTLM/Kerberos proxies. If you cannot reach GitHub at all, download `go-containerregistry_Windows_x86_64.tar.gz` from the [releases page](https://github.com/google/go-containerregistry/releases) on another machine and copy it over.

**Alternative (Scoop):** `scoop install crane`

---

## 2. Configure the proxy

### How crane finds the proxy

crane is a Go program. It reads the proxy **only from environment variables**:

| Variable | Used for |
|---|---|
| `HTTPS_PROXY` | All `https://` registry traffic. This is the one that matters for Docker Hub. |
| `HTTP_PROXY` | `http://` traffic (only with `--insecure` / plain-HTTP registries) |
| `NO_PROXY` | Comma-separated hosts that must **bypass** the proxy. Put your Harbor host here. |

> **Important:** crane does **not** use the Windows system proxy (Internet Options / *Settings → Network → Proxy*), WinHTTP settings, or PAC/WPAD auto-config scripts. Browsers and `Invoke-WebRequest` can work while crane still fails. You must set the variables above.

### Find your proxy address

If you don't know the proxy address, check what Windows uses:

```powershell
# Per-user (Internet Options) proxy
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
  Select-Object ProxyEnable, ProxyServer, ProxyOverride, AutoConfigURL

# Machine-wide WinHTTP proxy
netsh winhttp show proxy
```

- If `ProxyServer` is set (e.g. `proxy.corp.local:8080`), use it.
- If only `AutoConfigURL` (a PAC file) is set, open that URL in a browser and look for the `PROXY host:port` line used for external sites. You can also ask your network team.
- `ProxyOverride` lists the bypass hosts, which is a good starting point for `NO_PROXY`. Note that it uses `;` separators, while `NO_PROXY` uses `,`.

### Set the variables for the current session

```powershell
$env:HTTPS_PROXY = 'http://proxy.corp.local:8080'
$env:HTTP_PROXY  = 'http://proxy.corp.local:8080'
$env:NO_PROXY    = 'harbor.example.local,.example.local,localhost,127.0.0.1,10.0.0.0/8'
```

Notes:

- The proxy URL normally starts with `http://` even though it carries HTTPS traffic, because crane tunnels HTTPS through the proxy with `CONNECT`.
- **Harbor must be in `NO_PROXY`**, or crane will try to reach your internal Harbor through the internet proxy and fail. `NO_PROXY` entries can be:
  - a host (`harbor.example.local`, which also matches its subdomains)
  - a domain suffix (`.example.local`)
  - an IP address
  - a CIDR range (`10.0.0.0/8`)
  - a `host:port`
- Loopback addresses (`localhost`, `127.0.0.1`) are never proxied, whatever the settings.
- Environment variable names are case-insensitive on Windows, so `HTTPS_PROXY` and `https_proxy` are the same variable.

### Make it permanent (current user)

```powershell
[Environment]::SetEnvironmentVariable('HTTPS_PROXY', 'http://proxy.corp.local:8080', 'User')
[Environment]::SetEnvironmentVariable('HTTP_PROXY',  'http://proxy.corp.local:8080', 'User')
[Environment]::SetEnvironmentVariable('NO_PROXY',    'harbor.example.local,.example.local,localhost,127.0.0.1', 'User')
```

Open a **new** terminal afterwards. Be aware that other Go and CLI tools (git, kubectl, terraform, ...) will also pick these up.

### Proxy that needs a username and password (Basic auth)

Put the credentials in the URL. Special characters in the password (`@ : / # % ?` etc.) **must be URL-encoded**:

```powershell
$proxyUser = 'DOMAIN\jdoe'
$proxyPass = Read-Host 'Proxy password' -AsSecureString
$plain     = [System.Net.NetworkCredential]::new('', $proxyPass).Password
$u = [uri]::EscapeDataString($proxyUser)
$p = [uri]::EscapeDataString($plain)

$env:HTTPS_PROXY = "http://${u}:${p}@proxy.corp.local:8080"
$env:HTTP_PROXY  = $env:HTTPS_PROXY
```

Only set credentialed proxy URLs **per session**. Don't store them permanently with `SetEnvironmentVariable`, because user environment variables are stored in plain text.

### Proxy that uses Windows login (NTLM / Kerberos / "Negotiate")

crane (Go) **cannot** authenticate to NTLM/Kerberos proxies. If your browser works without asking for a password but crane gets `407 Proxy Authentication Required`, run a small local helper proxy that logs in with your Windows identity, then point crane at it:

- [Px](https://github.com/genotrance/px) (`pip install px-proxy`, then run `px`. It reads your Windows proxy/PAC settings and listens on `127.0.0.1:3128`.)
- [Cntlm](https://cntlm.sourceforge.net/) (listens on `127.0.0.1:3128` by default)

```powershell
$env:HTTPS_PROXY = 'http://127.0.0.1:3128'
$env:HTTP_PROXY  = 'http://127.0.0.1:3128'
```

### Hosts the proxy/firewall must allow for Docker Hub

| Host | Purpose |
|---|---|
| `index.docker.io`, `registry-1.docker.io` | Registry API / manifests |
| `auth.docker.io` | Token service |
| `production.cloudflare.docker.com` | Image layer (blob) downloads |

### Quick proxy test

```powershell
crane digest nginx:1.27
```

If it prints `sha256:...`, the proxy settings work. If not, add `-v` to see each HTTP request (`crane -v digest nginx:1.27`) and check [Troubleshooting](#10-troubleshooting).

---

## 3. Prepare Harbor

Harbor only accepts pushes into a **project** that already exists:

1. In the Harbor UI, open **Projects → New Project** and create one (for example `library`, which usually exists by default).
2. Make sure your user is a **Developer**, **Maintainer** or **Project Admin** on that project. The Guest and Limited Guest roles cannot push.
3. (Optional, for automation) Create a **Robot Account** with push permission and use its name and secret as the username and password below.

The image path in Harbor is always `<harbor-host>/<project>/<repository>:<tag>`.

---

## 4. Log in

### Harbor (required)

Interactive login with a hidden password prompt:

```powershell
$cred = Get-Credential -UserName 'myuser' -Message 'Harbor login'
$cred.GetNetworkCredential().Password | crane auth login harbor.example.local -u $cred.UserName --password-stdin
```

Non-interactive (CI / scheduled task), with the password in an environment variable:

```powershell
$env:HARBOR_PASSWORD | crane auth login harbor.example.local -u myuser --password-stdin
```

Avoid `-p 'mypassword'` on shared machines. The password ends up in PowerShell history (`(Get-PSReadLineOption).HistorySavePath`) and in the process list.

> **Robot accounts:** Harbor robot account names contain a `$` (e.g. `robot$ci-pusher`). In PowerShell, use **single quotes**, because double quotes expand `$ci` as a variable:
>
> ```powershell
> $env:ROBOT_SECRET | crane auth login harbor.example.local -u 'robot$ci-pusher' --password-stdin
> ```

> **Non-ASCII passwords on Windows PowerShell 5.1:** piping to a native program uses ASCII by default, which garbles characters like `é` or `ü`. Run `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)` first. PowerShell 7 already uses UTF-8.

### Docker Hub (optional, recommended)

Anonymous pulls work but are rate-limited. Logging in raises the limit:

```powershell
$env:DOCKERHUB_TOKEN | crane auth login index.docker.io -u mydockerhubuser --password-stdin
```

### Where credentials are stored

`crane auth login` uses the standard Docker config, `%USERPROFILE%\.docker\config.json`:

- If that file has a `"credsStore"` entry (for example `"desktop"` from Docker Desktop, or `"wincred"`), crane saves the password in **Windows Credential Manager** through that helper. The matching `docker-credential-<name>.exe` must be on `PATH`.
- Otherwise the credentials are saved base64-encoded (**not encrypted**) inside `config.json`.

To keep crane's credentials separate (useful on shared or build machines), point `DOCKER_CONFIG` at a private folder:

```powershell
$env:DOCKER_CONFIG = Join-Path $env:TEMP "crane-$(New-Guid)"
New-Item -ItemType Directory -Path $env:DOCKER_CONFIG | Out-Null
$env:HARBOR_PASSWORD | crane auth login harbor.example.local -u myuser --password-stdin
# ... crane commands ...
Remove-Item -Recurse -Force $env:DOCKER_CONFIG
```

To log out:

```powershell
crane auth logout harbor.example.local
```

---

## 5. Option A: copy directly from Docker Hub to Harbor (recommended)

`crane copy` (alias `crane cp`) streams the image from Docker Hub (through the proxy) to Harbor (direct, via `NO_PROXY`). It writes nothing to disk, keeps **all architectures** of a multi-arch image, and preserves the digest.

```powershell
crane copy docker.io/library/nginx:1.27 harbor.example.local/library/nginx:1.27
```

More examples:

```powershell
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

> Option A requires a machine that can reach **both** the proxy (for Docker Hub) and Harbor. If no single machine can, use Option B.

---

## 6. Option B: pull to a file, then push

Use this when the internet-facing machine cannot reach Harbor, or when you want an offline copy.

### Step 1: pull (machine with proxy/internet access)

```powershell
New-Item -ItemType Directory -Force C:\images | Out-Null

# Single platform -> tarball
crane pull --platform linux/amd64 nginx:1.27 C:\images\nginx_1.27_amd64.tar
```

To keep a **multi-arch** image intact, save it as an OCI image layout folder, then zip it for transfer:

```powershell
crane pull --format=oci nginx:1.27 C:\images\nginx-oci
Compress-Archive -Path C:\images\nginx-oci -DestinationPath C:\images\nginx-oci.zip
```

### Step 2: move the file

Copy the `.tar` or `.zip` file to the machine that can reach Harbor, using a file share, USB storage or similar.

### Step 3: push (machine that can reach Harbor)

Log in to Harbor first (see [section 4](#4-log-in)). Make sure Harbor is not routed through a proxy: either clear the proxy variables or include Harbor in `NO_PROXY`.

```powershell
# Single-image tarball
crane push C:\images\nginx_1.27_amd64.tar harbor.example.local/library/nginx:1.27

# Multi-arch OCI layout: every platform is kept and the digest matches Docker Hub
Expand-Archive C:\images\nginx-oci.zip -DestinationPath C:\images
crane push C:\images\nginx-oci harbor.example.local/library/nginx:1.27
```

> Do **not** add `--index` when you push a single multi-arch OCI layout. It wraps the original index inside a new one, which changes the digest and can confuse some clients. `--index` is only needed when a path holds several separate images, and even then it combines them into **one** index under a single tag. To push images under separate names, pull each one into its own file.

---

## 7. Verify the upload

```powershell
# List tags in the Harbor repository
crane ls harbor.example.local/library/nginx

# Compare digests between source and destination (they match for crane copy)
crane digest nginx:1.27
crane digest harbor.example.local/library/nginx:1.27

# Inspect the manifest (shows the platform list for a multi-arch image)
crane manifest harbor.example.local/library/nginx:1.27 | ConvertFrom-Json | ConvertTo-Json -Depth 10

# Inspect the image config (env, entrypoint, labels, ...)
crane config harbor.example.local/library/nginx:1.27 | ConvertFrom-Json | ConvertTo-Json -Depth 10
```

You can also confirm in the Harbor UI under **Projects → library → Repositories**.

---

## 8. Mirror a list of images (PowerShell script)

Create `images.txt` with one Docker Hub image per line (`#` lines are ignored):

```text
nginx:1.27
redis:7.4
postgres:16
bitnami/kafka:3.7
```

Save this as `Mirror-Images.ps1`:

```powershell
[CmdletBinding()]
param(
    [string]$ListFile  = '.\images.txt',
    [string]$Harbor    = 'harbor.example.local',
    [string]$Project   = 'library',
    [string]$Proxy     = $env:HTTPS_PROXY,
    [string]$NoProxy   = $null,
    [pscredential]$Credential = (Get-Credential -Message "Harbor login for $Harbor")
)

$ErrorActionPreference = 'Stop'

if ($Proxy) {
    $env:HTTPS_PROXY = $Proxy
    $env:HTTP_PROXY  = $Proxy
}
# Harbor must always bypass the proxy
$env:NO_PROXY = (@($Harbor, 'localhost', '127.0.0.1') + ($NoProxy -split ',') |
                 Where-Object { $_ } | Select-Object -Unique) -join ','

$Credential.GetNetworkCredential().Password |
    crane auth login $Harbor -u $Credential.UserName --password-stdin
if ($LASTEXITCODE -ne 0) { throw "Login to $Harbor failed" }

$failed = @()
Get-Content $ListFile | ForEach-Object { $_.Trim() } |
  Where-Object { $_ -and -not $_.StartsWith('#') } |
  ForEach-Object {
    $src = $_
    # "bitnami/kafka:3.7" -> "kafka:3.7"; "nginx:1.27" -> "nginx:1.27"
    $nameTag = ($src -split '/')[-1]
    $dst = "$Harbor/$Project/$nameTag"
    Write-Host ">> $src -> $dst" -ForegroundColor Cyan
    crane copy $src $dst
    if ($LASTEXITCODE -ne 0) { $failed += $src }
  }

if ($failed) {
    Write-Warning "Failed: $($failed -join ', ')"
    exit 1
}
Write-Host 'All images mirrored.' -ForegroundColor Green
```

Run it:

```powershell
# If scripts are blocked: Set-ExecutionPolicy -Scope Process Bypass
.\Mirror-Images.ps1 -ListFile .\images.txt -Harbor harbor.example.local -Project library `
                    -Proxy 'http://proxy.corp.local:8080'
```

For unattended runs, pass a credential object instead of being prompted:

```powershell
$sec  = ConvertTo-SecureString $env:HARBOR_PASSWORD -AsPlainText -Force
$cred = [pscredential]::new('robot$ci-pusher', $sec)
.\Mirror-Images.ps1 -Credential $cred -Proxy 'http://proxy.corp.local:8080'
```

---

## 9. Certificates: Harbor CA and TLS-inspecting proxies

On Windows, crane checks TLS certificates against the **Windows certificate store**. The `SSL_CERT_FILE` and `SSL_CERT_DIR` variables used on Linux are **ignored on Windows**.

### Harbor uses a self-signed or internal CA

Import the CA certificate (not the server certificate) into Trusted Root:

```powershell
# Current user only (no admin needed)
Import-Certificate -FilePath .\harbor-ca.crt -CertStoreLocation Cert:\CurrentUser\Root

# All users (run PowerShell as Administrator)
Import-Certificate -FilePath .\harbor-ca.crt -CertStoreLocation Cert:\LocalMachine\Root
```

You can usually download the CA certificate from the Harbor UI: **Projects → (project) → Registry Certificate**, or **Administration → Configuration → System Settings**.

### Proxy does TLS inspection (SSL interception)

Many corporate proxies re-sign HTTPS traffic with their own root CA. If that root CA is already deployed to Windows (usually by Group Policy), crane trusts it automatically. If you see `x509: certificate signed by unknown authority` only for Docker Hub, ask IT for the proxy's root CA and import it the same way as above.

### Plain-HTTP Harbor or skipping verification (lab use only)

Add the global `--insecure` flag to every command that touches Harbor:

```powershell
crane auth login --insecure harbor.example.local:8080 -u myuser -p 'mypassword'
crane copy --insecure nginx:1.27 harbor.example.local:8080/library/nginx:1.27
```

If Harbor listens on a port other than 443, include the port in **every** reference (login, image names and `NO_PROXY`). The credential entry is keyed by `host:port`.

---

## 10. Troubleshooting

Add `-v` to any crane command to print every HTTP request, including which host it is calling:

```powershell
crane -v copy nginx:1.27 harbor.example.local/library/nginx:1.27 2>&1 | Tee-Object crane-debug.log
```

| Symptom | Likely cause and fix |
|---|---|
| `dial tcp ...:443: i/o timeout` or `connectex: A connection attempt failed` when talking to `index.docker.io` | No proxy configured. crane ignores the Windows system proxy. Set `HTTPS_PROXY` ([section 2](#2-configure-the-proxy)). |
| `proxyconnect tcp: dial tcp <proxy>: connect: connection refused` / `no such host` | Wrong proxy host or port, or the helper proxy (Px/Cntlm) isn't running. |
| `Proxy Authentication Required` / `407` | The proxy needs credentials. Use `http://user:pass@proxy:port` (URL-encoded), or Px/Cntlm for NTLM/Kerberos proxies. |
| Pulls from Docker Hub work, but Harbor gives `timeout`, `403`, `502` or `Bad Gateway` | Harbor traffic is going through the internet proxy. Add the Harbor host (and `:port` if used) to `NO_PROXY` and open a new terminal if you set it permanently. |
| `x509: certificate signed by unknown authority` | Harbor uses an internal CA, or the proxy does TLS inspection. Import the CA into the Windows store ([section 9](#9-certificates-harbor-ca-and-tls-inspecting-proxies)). |
| `http: server gave HTTP response to HTTPS client` | Harbor is serving plain HTTP. Add `--insecure`. |
| `UNAUTHORIZED: unauthorized to access repository` | Not logged in to that exact `host[:port]`, or wrong password. Run `crane auth login` again with the same host string as in the image reference. |
| `DENIED: requested access to the resource is denied` / `project not found` | The Harbor project doesn't exist, or your user/robot lacks push permission ([section 3](#3-prepare-harbor)). |
| Robot account login fails | `$` in `robot$name` was expanded because of double quotes. Use single quotes. |
| `error getting credentials` / `docker-credential-desktop.exe: executable file not found` | `config.json` names a credential helper that isn't installed. Use a separate `DOCKER_CONFIG` folder ([section 4](#4-log-in)) or remove `"credsStore"` from `config.json`. |
| `TOOMANYREQUESTS: You have reached your pull rate limit` | Docker Hub anonymous limit (often shared by the whole office behind one proxy IP). Log in to `index.docker.io`. |
| Only one architecture shows up in Harbor | You pulled to a tarball or used `--platform`. Use `crane copy`, or `crane pull --format=oci` followed by `crane push <folder>`. |
| `crane : The term 'crane' is not recognized` | `crane.exe` isn't on `PATH`. Open a new terminal after installing, or run it by full path. |

---

## 11. Command cheat sheet

```powershell
# Proxy (session only)
$env:HTTPS_PROXY = 'http://proxy.corp.local:8080'
$env:HTTP_PROXY  = 'http://proxy.corp.local:8080'
$env:NO_PROXY    = 'harbor.example.local,localhost,127.0.0.1'

# Auth
$env:HARBOR_PASSWORD | crane auth login harbor.example.local -u myuser --password-stdin
crane auth logout harbor.example.local

# Registry to registry (all archs, no disk)
crane copy nginx:1.27 harbor.example.local/library/nginx:1.27

# Two-step via tarball (single arch)
crane pull --platform linux/amd64 nginx:1.27 C:\images\nginx.tar
crane push C:\images\nginx.tar harbor.example.local/library/nginx:1.27

# Two-step, keeping multi-arch
crane pull --format=oci nginx:1.27 C:\images\nginx-oci
crane push C:\images\nginx-oci harbor.example.local/library/nginx:1.27

# Inspect
crane ls       harbor.example.local/library/nginx
crane digest   harbor.example.local/library/nginx:1.27
crane manifest harbor.example.local/library/nginx:1.27
crane config   harbor.example.local/library/nginx:1.27

# Retag inside Harbor (no data transfer)
crane tag harbor.example.local/library/nginx:1.27 stable

# Delete a manifest (requires delete permission in Harbor)
crane delete harbor.example.local/library/nginx@sha256:<digest>
```

Full crane reference: <https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md>
