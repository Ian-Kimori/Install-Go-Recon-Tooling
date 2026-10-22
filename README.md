# Install-Golang-Subfinder-Amass-and-Katana

Here's the full setup with katana added.

## 1. Clear the dead proxy

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy
```
```bash
grep -rn -i proxy /etc/environment ~/.bashrc ~/.zshrc ~/.profile /etc/apt/apt.conf.d/ 2>/dev/null
```

Delete any line pointing at `172.234.27.37:8080` (use `sudo nano` for files under `/etc`). Then open a new terminal and confirm it's gone:

```bash
env | grep -i proxy    # should print nothing
```

## 2. Install Go

Use `golang-go`, not `gccgo-go`, which is too old and conflicts with it. This also pulls in `gcc`, which katana needs:

```bash
sudo apt remove -y gccgo-go 2>/dev/null
sudo apt clean
sudo apt update
```
```bash
sudo apt install -y golang-go gcc
go version    # should show go1.26.x, not gccgo
```

If you get a 404, the mirror is mid-sync. Wait a few minutes and rerun `sudo apt update`.

## 3. Put Go's bin directory on PATH

Use `~/.zshrc` or any other instead if `echo $SHELL` shows zsh or any:

```bash
echo $SHELL
```

```bash
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
```
```bash
source ~/.bashrc
```
```bash
go env -u HTTP_PROXY HTTPS_PROXY 2>/dev/null
```

## 4. Install the tools

```bash
# subfinder
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# amass (v5)
CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main

# katana (needs CGO, so gcc must be installed)
CGO_ENABLED=1 go install github.com/projectdiscovery/katana/cmd/katana@latest
```

Skip `apt install amass` and the snap package, since both carry older v3/v4 builds.

## If `go install` times out

Fetch straight from GitHub instead of the Go module proxy:

```bash
GOPROXY=direct go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
CGO_ENABLED=0 GOPROXY=direct go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
CGO_ENABLED=1 GOPROXY=direct go install github.com/projectdiscovery/katana/cmd/katana@latest
```

## Fallback: prebuilt binaries, no Go needed

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
```

## 5. Optional: headless crawling for katana

Katana's `-headless` mode, used for JavaScript-heavy sites, needs a Chromium-based browser. It can download one itself on first use, or you can install it:

```bash
sudo apt install -y chromium-browser
```

## 6. Verify and configure

```bash
which subfinder amass katana
subfinder -version
amass -h
katana -version
```

API keys make a big difference to how many subdomains you find:

- **subfinder:** `~/.config/subfinder/provider-config.yaml`, created on first run.
- **amass:** `~/.config/amass/datasources.yaml`.

Quick test:

```bash
subfinder -d example.com -silent
amass enum -d example.com
katana -u https://example.com -jc

# chained: find subdomains, then crawl them
subfinder -d example.com -silent | katana -silent
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `proxyconnect tcp ... i/o timeout` | Proxy env var still set. Redo step 1 in a fresh terminal. |
| `404 Not Found` from apt | Stale index. Run `sudo apt update`, or wait for the mirror to sync. |
| `command not found` after install | PATH missing Go's bin. Redo step 3, or run `~/go/bin/<tool>`. |
| `requires go >= 1.xx` | Go is too old or gccgo is installed. Redo step 2. |
| `gcc: not found` (katana) | Run `sudo apt install -y gcc`. |
| `proxy.golang.org` timeout | Prefix the install with `GOPROXY=direct`. |
