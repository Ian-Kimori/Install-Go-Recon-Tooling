# Install Go + Recon Tooling (Subfinder · Amass · Katana · gau · waybackurls) — Every Linux Distro

# Install Go + Recon Tooling (Subfinder · Amass · Katana · gau · waybackurls · httpx · trufflehog) — Every Linux Distro

Full setup for the Go-based recon stack — **subfinder** and **amass**
(subdomain enumeration), **katana** (active crawling), **gau** and
**waybackurls** (passive URL collection from the Wayback Machine, Common Crawl,
AlienVault OTX, and friends), **httpx** (fast HTTP probing — which hosts are
live, plus their status codes, titles, and tech fingerprints), and
**trufflehog** (secrets scanning across git repos, GitHub/GitLab orgs,
filesystems, and URLs) — extended to cover **every major Linux
distribution**: Debian/Ubuntu (and derivatives like Kali, Mint, Pop!_OS),
Fedora, RHEL/CentOS/Rocky/AlmaLinux, Arch/Manjaro, openSUSE, Alpine, Void,
Gentoo, and NixOS.

Most of this guide is distro-independent. Only four things actually differ
between distros: (1) where proxy settings hide in your package-manager config,
(2) how you install Go and a C compiler, (3) how you install Chromium for
headless crawling, and (4) a handful of troubleshooting fixes. Everything
else — PATH setup, `go install`, the prebuilt-binary fallback, and
verification — is identical everywhere. `httpx` and `trufflehog` are both pure
Go and need no C compiler, so they add nothing distro-specific.

> **Section map**
> - Steps **3, 4, "If go install times out", "Fallback binaries"**, and
>   **6 (verify)** are the same on all distros — run them as written.
> - Steps **1 (proxy)**, **2 (install Go)**, and **5 (Chromium)** have a
>   per-distro variant below. Find your distro's block and use that.

---

## 1. Clear the dead proxy

The shell part is **identical on every distro**:

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy
```
```bash
grep -rn -i proxy /etc/environment ~/.bashrc ~/.zshrc ~/.profile 2>/dev/null
```

Delete any line pointing at `172.234.27.37:8080` (use `sudo nano` for files
under `/etc`). Then open a new terminal and confirm it's gone:

```bash
env | grep -i proxy    # should print nothing
```

### Also check your package manager's own proxy config

A proxy can also be baked into the package manager itself, which is what breaks
`apt update` / `dnf update` even after the env vars are cleared. Check the file
for your distro and delete any dead-proxy line:

| Distro family | Package-manager proxy lives in |
|---|---|
| Debian / Ubuntu / Kali / Mint | `/etc/apt/apt.conf.d/` (any file with `Acquire::http::Proxy`) |
| Fedora / RHEL / Rocky / Alma | `/etc/dnf/dnf.conf` (`proxy=` line) |
| CentOS 7 (yum) | `/etc/yum.conf` (`proxy=` line) |
| Arch / Manjaro | pacman has no proxy config — it reads `http_proxy`/`https_proxy` env vars only |
| openSUSE | `/etc/sysconfig/proxy` and `/etc/zypp/zypp.conf`; also `PROXY_ENABLED` in `/etc/sysconfig/proxy` |
| Alpine | apk reads `http_proxy`/`https_proxy` env vars only (no separate config) |
| Void | xbps reads env vars; also check `/etc/xbps.d/` |
| Gentoo | `/etc/portage/make.conf` (`http_proxy`/`https_proxy` lines) |
| NixOS | `networking.proxy.*` in `/etc/nixos/configuration.nix` |

Generalized grep across the common spots:

```bash
grep -rn -i proxy /etc/apt/apt.conf.d/ /etc/dnf/dnf.conf /etc/yum.conf \
  /etc/zypp/zypp.conf /etc/sysconfig/proxy /etc/portage/make.conf \
  /etc/xbps.d/ 2>/dev/null
```

---

## 2. Install Go

subfinder, amass v5, and katana need a **recent** Go toolchain (roughly Go
1.23+). Many distro packages ship an older Go, so there are two routes:

- **Method A (universal, recommended):** official Go tarball. Same on every
  distro, always current, and sidesteps every "requires go >= 1.xx" problem.
- **Method B (per-distro package manager):** convenient, but only use it if
  your distro ships a new enough Go (Arch, Fedora, openSUSE Tumbleweed, Alpine
  edge, Void, Gentoo, NixOS are generally fine; Debian stable, Ubuntu LTS,
  RHEL/CentOS, and amass v5 together are often **too old**).

You still need a **C compiler (`gcc`)** regardless of method, because katana is
built with `CGO_ENABLED=1`. (httpx, trufflehog, gau, and waybackurls do **not**
need it — they're pure Go.)

### Method A — Official Go tarball (works on all distros)

```bash
cd /tmp

# fetch the latest version string (e.g. go1.26.0)
GO_VERSION=$(curl -s https://go.dev/VERSION?m=text | head -1)

# amd64; on ARM boards swap linux-amd64 -> linux-arm64
wget https://go.dev/dl/${GO_VERSION}.linux-amd64.tar.gz

sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf ${GO_VERSION}.linux-amd64.tar.gz

export PATH=$PATH:/usr/local/go/bin
go version    # should show the version you just installed
```

Make the PATH entry permanent (and still do **step 3** for the tool bin dir):

```bash
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc   
source ~/.bashrc
```
**or ~/.zshrc**
```bash
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.zshrc
source ~/.zshrc
```

You still need gcc — install it with your distro's command from Method B below
(just the `gcc`/build-tools part).

### Method B — Per-distro package manager

#### Debian / Ubuntu / Kali / Mint / Pop!_OS (apt)

Use `golang-go`, not `gccgo-go`, which is too old and conflicts with it. This
also pulls in `gcc`, which katana needs:

```bash
sudo apt remove -y gccgo-go 2>/dev/null
sudo apt clean
sudo apt update
```
```bash
sudo apt install -y golang-go gcc
go version    # should show go1.26.x, not gccgo
```

If you get a 404, the mirror is mid-sync. Wait a few minutes and rerun
`sudo apt update`.

> Debian stable / Ubuntu LTS often ship an older Go. If `go version` is below
> what amass v5 or katana demand, use **Method A** instead.

#### Fedora (dnf)

```bash
sudo dnf install -y golang gcc
go version
```

Fedora usually ships a current Go, so Method B is fine here.

#### RHEL / CentOS Stream / Rocky / AlmaLinux (dnf)

```bash
sudo dnf install -y golang gcc
go version
```

> RHEL-family Go can lag. If it's too old, use **Method A**. Chromium (step 5)
> needs EPEL enabled:
> ```bash
> sudo dnf install -y epel-release
> ```

#### CentOS 7 (yum)

```bash
sudo yum install -y golang gcc
go version
```

> CentOS 7's `golang` is very old (1.x series) and will almost certainly be
> rejected by amass/katana. Prefer **Method A** on CentOS 7.

#### Arch / Manjaro / EndeavourOS (pacman)

```bash
sudo pacman -S --needed go gcc
go version
```

Arch tracks upstream closely, so this is always current.

#### openSUSE (Leap / Tumbleweed — zypper)

```bash
sudo zypper install -y go gcc
go version
```

> On openSUSE Leap the `go` meta-package may point at an older release; if so,
> install a versioned package (e.g. `go1.24`) or use **Method A**. Tumbleweed
> is current.

#### Alpine (apk)

Alpine uses **musl**, not glibc, so CGO builds (katana) need `musl-dev`:

```bash
sudo apk add go gcc musl-dev
go version
```

> If `go` isn't found, enable the community repo in `/etc/apk/repositories`,
> then `sudo apk update`.

#### Void Linux (xbps)

```bash
sudo xbps-install -S go gcc
go version
```

#### Gentoo (emerge)

```bash
sudo emerge --ask dev-lang/go
# gcc is part of the base system; install explicitly only if missing:
sudo emerge --ask sys-devel/gcc
go version
```

#### NixOS / Nix (nix-shell)

Imperative, per-shell (gives you go **and** gcc for CGO in one shot):

```bash
nix-shell -p go gcc
go version
```

Or declaratively, add to `/etc/nixos/configuration.nix`:

```nix
environment.systemPackages = with pkgs; [ go gcc ];
```
then `sudo nixos-rebuild switch`.

---

## 3. Put Go's bin directory on PATH

**Same on every distro.** Use `~/.zshrc` or any other instead if `echo $SHELL`
shows zsh or any:

```bash
echo $SHELL
```

```bash
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
```

> If you installed Go via **Method A (tarball)**, make sure both
> `/usr/local/go/bin` (the toolchain) **and** `$(go env GOPATH)/bin` (your
> installed tools) are on PATH.

---

## 4. Install the tools

**Same on every distro** once Go and gcc are in place:

```bash
# subfinder
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# amass (v5)
CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main

# katana (needs CGO, so gcc must be installed)
CGO_ENABLED=1 go install github.com/projectdiscovery/katana/cmd/katana@latest

# gau (getallurls) — pure Go, no CGO
go install -v github.com/lc/gau/v2/cmd/gau@latest

# waybackurls — pure Go, no CGO
go install -v github.com/tomnomnom/waybackurls@latest

# httpx (ProjectDiscovery HTTP prober) — pure Go, no CGO
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest

# trufflehog (secrets scanner) — pure Go, no CGO
go install github.com/trufflesecurity/trufflehog/v3@latest
```

Skip your distro's packaged `amass` (apt/dnf/pacman/etc.) and the snap package,
since both carry older v3/v4 builds. gau, waybackurls, httpx, and trufflehog are
pure-Go tools and need no C compiler, so they build fine on any distro —
including Alpine — without `musl-dev`.

> **httpx naming clash:** ProjectDiscovery's `httpx` shares its name with the
> Python `httpx` library and, on some distros, a differently-packaged binary
> (Kali ships the ProjectDiscovery tool as `httpx-toolkit`). After installing,
> run `which httpx` and confirm it points at `$(go env GOPATH)/bin/httpx` — not
> a `/usr/bin/httpx` left behind by a `pip install httpx`. Prefer `go install`
> or the prebuilt binary over distro packages for this one.
>
> **trufflehog without Go:** trufflehog pulls in a lot of dependencies, so
> `go install` can be slow. The official one-line install script (see the
> fallback section) grabs the right prebuilt binary and needs no Go toolchain
> at all.

---

## If `go install` times out

**Same on every distro.** Fetch straight from GitHub instead of the Go module
proxy:

```bash
GOPROXY=direct go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
CGO_ENABLED=0 GOPROXY=direct go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
CGO_ENABLED=1 GOPROXY=direct go install github.com/projectdiscovery/katana/cmd/katana@latest
GOPROXY=direct go install -v github.com/lc/gau/v2/cmd/gau@latest
GOPROXY=direct go install -v github.com/tomnomnom/waybackurls@latest
GOPROXY=direct go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
GOPROXY=direct go install github.com/trufflesecurity/trufflehog/v3@latest
```

---

## Fallback: prebuilt binaries, no Go needed

**Same on every distro** (glibc). These are statically linked Go binaries:

```bash
cd /tmp

# subfinder
V=$(curl -s https://api.github.com/repos/projectdiscovery/subfinder/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/projectdiscovery/subfinder/releases/download/v${V}/subfinder_${V}_linux_amd64.zip
unzip -o subfinder_${V}_linux_amd64.zip subfinder && sudo mv subfinder /usr/local/bin/

# katana
V=$(curl -s https://api.github.com/repos/projectdiscovery/katana/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/projectdiscovery/katana/releases/download/v${V}/katana_${V}_linux_amd64.zip
unzip -o katana_${V}_linux_amd64.zip katana && sudo mv katana /usr/local/bin/

# amass
wget https://github.com/owasp-amass/amass/releases/latest/download/amass_Linux_amd64.zip
unzip -o amass_Linux_amd64.zip && sudo mv amass_Linux_amd64/amass /usr/local/bin/

# gau
V=$(curl -s https://api.github.com/repos/lc/gau/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/lc/gau/releases/download/v${V}/gau_${V}_linux_amd64.tar.gz
tar -xzf gau_${V}_linux_amd64.tar.gz gau && sudo mv gau /usr/local/bin/

# waybackurls (rarely re-released; go install is the reliable route)
V=$(curl -s https://api.github.com/repos/tomnomnom/waybackurls/releases/latest | grep -oP '"tag_name": "v?\K[^"]+')
wget https://github.com/tomnomnom/waybackurls/releases/download/v${V}/waybackurls-linux-amd64-${V}.tgz
tar -xzf waybackurls-linux-amd64-${V}.tgz waybackurls && sudo mv waybackurls /usr/local/bin/

# httpx
V=$(curl -s https://api.github.com/repos/projectdiscovery/httpx/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/projectdiscovery/httpx/releases/download/v${V}/httpx_${V}_linux_amd64.zip
unzip -o httpx_${V}_linux_amd64.zip httpx && sudo mv httpx /usr/local/bin/

# trufflehog — official install script (no Go needed; grabs the right prebuilt binary)
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sudo sh -s -- -b /usr/local/bin

# trufflehog — manual, if you'd rather not pipe a script to sh
V=$(curl -s https://api.github.com/repos/trufflesecurity/trufflehog/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/trufflesecurity/trufflehog/releases/download/v${V}/trufflehog_${V}_linux_amd64.tar.gz
tar -xzf trufflehog_${V}_linux_amd64.tar.gz trufflehog && sudo mv trufflehog /usr/local/bin/
```

> **Alpine / musl note:** these glibc-targeted binaries may not run directly on
> Alpine. The pure-Go ones (subfinder, amass with `CGO_ENABLED=0`, gau,
> waybackurls, httpx, trufflehog) usually work because they're static; if a
> binary fails to start, either install the glibc shim `sudo apk add gcompat`,
> or build from source as in step 4 (with `musl-dev` installed).
>
> **ARM note:** swap `linux_amd64` / `Linux_amd64` for the arm64 asset name on
> ARM hardware.

---

## 5. Optional: headless crawling for katana

Katana's `-headless` mode, used for JavaScript-heavy sites, needs a
Chromium-based browser. It can download one itself on first use, or you can
install it with your distro's package manager. (httpx and trufflehog do **not**
need a browser.)

#### Debian / Ubuntu / Kali / Mint

```bash
sudo apt install -y chromium-browser    # Ubuntu; on Debian the package is 'chromium'
```

#### Fedora

```bash
sudo dnf install -y chromium
```

#### RHEL / CentOS / Rocky / Alma (needs EPEL — see step 2)

```bash
sudo dnf install -y epel-release
sudo dnf install -y chromium
```

#### Arch / Manjaro

```bash
sudo pacman -S chromium
```

#### openSUSE

```bash
sudo zypper install -y chromium
```

#### Alpine

```bash
sudo apk add chromium chromium-chromedriver nss freetype harfbuzz ca-certificates ttf-freefont
```

#### Void

```bash
sudo xbps-install -S chromium
```

#### Gentoo

```bash
sudo emerge --ask www-client/chromium    # compiles for hours; 'google-chrome' binary pkg is faster
```

#### NixOS

```bash
nix-shell -p chromium
# or add 'chromium' to environment.systemPackages in configuration.nix
```

---

## 6. Verify and configure

**Same on every distro.**

```bash
which subfinder amass katana gau waybackurls httpx trufflehog
subfinder -version
amass -h
katana -version
gau --version
waybackurls -h       # prints usage; no version flag
httpx -version
trufflehog --version
```

API keys make a big difference to how many subdomains you find:

- **subfinder:** `~/.config/subfinder/provider-config.yaml`, created on first run.
- **amass:** `~/.config/amass/datasources.yaml`.
- **gau:** optional `~/.gau.toml` (or `--config`) for OTX / URLScan keys and
  provider selection; works without any config too.
- **waybackurls:** no config or keys — pulls straight from the Wayback Machine.
- **httpx:** no config file or API keys — it just probes. Behavior is driven by
  flags (`-title`, `-status-code`, `-tech-detect`, `-follow-redirects`,
  `-json`, etc.).
- **trufflehog:** no API keys required to run. For GitHub/GitLab scans — private
  repos, or just to avoid rate limits — set a token via the `GITHUB_TOKEN`
  (or `GITLAB_TOKEN`) env var, or `--token`. Custom/extra detectors go in a
  YAML passed with `--config`.

Quick test:

```bash
subfinder -d example.com -silent
amass enum -d example.com
katana -u https://example.com -jc
gau example.com
waybackurls example.com
httpx -u https://example.com
trufflehog git https://github.com/trufflesecurity/test_keys   # known-positive test repo

# chained: find subdomains, then crawl them
subfinder -d example.com -silent | katana -silent

# chained: find subdomains, then pull every known URL for them (passive)
subfinder -d example.com -silent | gau
subfinder -d example.com -silent | waybackurls

# combine passive sources, dedupe, and feed into katana for active crawling
(echo example.com | gau; echo example.com | waybackurls) | sort -u | katana -silent

# chained: find subdomains, then keep only the ones that are actually live
subfinder -d example.com -silent | httpx -silent

# probe live hosts with status code, page title, and tech fingerprint
subfinder -d example.com -silent | httpx -silent -status-code -title -tech-detect

# scan a GitHub repo or whole org for leaked secrets
#   (set GITHUB_TOKEN first for private repos / higher rate limits)
trufflehog github --repo=https://github.com/example/repo
trufflehog github --org=example-org

# scan a local checkout's full git history, or just the working tree
trufflehog git file://$(pwd)
trufflehog filesystem /path/to/code

# show only verified (confirmed-live) secrets
trufflehog git https://github.com/example/repo --results=verified   # older versions: --only-verified
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `proxyconnect tcp ... i/o timeout` | Proxy env var still set. Redo step 1 in a fresh terminal (and check your package manager's proxy config). |
| `404 Not Found` from apt | Stale index. Run `sudo apt update`, or wait for the mirror to sync. |
| `404 Not Found` / stale metadata (dnf/zypp/pacman) | `sudo dnf clean all && sudo dnf makecache` / `sudo zypper refresh` / `sudo pacman -Syy`. |
| `command not found` after install | PATH missing Go's bin. Redo step 3, or run `~/go/bin/<tool>`. |
| `requires go >= 1.xx` | Go is too old or gccgo is installed. Use **Method A** (tarball) in step 2, or on Debian/Ubuntu redo the apt steps. |
| `gcc: not found` (katana) | Install a C compiler: `apt/dnf/zypp install gcc`, `pacman -S gcc`, `apk add gcc musl-dev`, etc. |
| `proxy.golang.org` timeout | Prefix the install with `GOPROXY=direct`. |
| Alpine: built binary won't start | Build from source with `musl-dev` installed, or `apk add gcompat` for glibc binaries. |
| RHEL/CentOS: `chromium` has no match | Enable EPEL first: `sudo dnf install -y epel-release`. |
| katana `-headless` can't find a browser | Install Chromium (step 5), or let katana auto-download on first `-headless` run. |
| `httpx` runs but behaves like the Python tool / wrong flags | Naming clash. Check `which httpx` points at `$(go env GOPATH)/bin/httpx`; remove any pip `httpx`, or on Kali use the `httpx-toolkit` package. |
| trufflehog `go install` is very slow or times out | It has many dependencies — use the official install script or the prebuilt binary (fallback section) instead. |
| trufflehog GitHub scan hits rate limits / can't see private repos | Export `GITHUB_TOKEN` (or pass `--token`) before scanning. |
