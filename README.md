# useful

Offline transfer artifacts for air-gapped machines.

## Install Claude Code on an air-gapped Kali Linux machine

Binary: `claude-linux-x64-2.1.278` — attached on the
[claude-code-2.1.278 release](https://github.com/chihkeong/useful/releases/tag/claude-code-2.1.278).

```bash
wget https://github.com/chihkeong/useful/releases/download/claude-code-2.1.278/claude-linux-x64-2.1.278
chmod +x claude-linux-x64-2.1.278
sudo mv claude-linux-x64-2.1.278 /usr/local/bin/claude   # or ~/.local/bin/claude (must be on PATH)
claude --version   # should print 2.1.278
```

Notes:
- The binary is dynamically linked (`ld-linux-x86-64.so.2`) — standard Kali x86-64
  libraries are sufficient; no runtime install needed.
- First run needs either an API key / login session set up beforehand, or whatever
  credentials your air-gap policy allows (e.g. a proxy endpoint via environment
  variables such as `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN`).

## Install Playwright Chromium on an air-gapped Kali Linux machine

Target: Playwright Chromium 151.0.7922.34 (browser build **1234**) + headless shell + FFmpeg (build **1011**), linux64.

The browser zips are attached as assets on the
[playwright-browsers release](https://github.com/chihkeong/useful/releases/tag/playwright-browsers)
(not committed to git — GitHub rejects files over 100 MB in a normal push).

### Integrity verification (verify after download on the target machine)

The Chromium zips were cross-verified against both Playwright's CDN and Google's
Chrome for Testing bucket (`storage.googleapis.com/chrome-for-testing-public`);
FFmpeg against Playwright's CDN only (it is a Playwright-built binary with no
second official source).

```bash
cat > SHA256SUMS <<'EOF'
ae8736ac28bc69278551500f219fc749575648263c43ec5990749eff43b9fcf8  chrome-linux64.zip
3cfc2bd00d1bafcf8a68dc74c9c92bb7150ddc8d26ade948a776316e1cec4f14  chrome-headless-shell-linux64.zip
ebc74fc5b94830176a3c2914ae96bd8bc7f6a91f4f33890230f84a172ee61ccc  ffmpeg-linux.zip
5c4735937844e84f8a93306e841a5b0e12252909b07870f789b190468da147ab  claude-linux-x64-2.1.278
EOF
sha256sum -c SHA256SUMS
```

All four lines must print `OK` before installing.

### Step 1 — Download the zips (internet-connected side)

```bash
mkdir -p ~/pw_transfer && cd ~/pw_transfer
BASE=https://github.com/chihkeong/useful/releases/download/playwright-browsers
wget $BASE/chrome-linux64.zip
wget $BASE/chrome-headless-shell-linux64.zip
wget $BASE/ffmpeg-linux.zip
```

### Step 2 — Verify against your Playwright version

The folder revision numbers below (`1234`, `1011`) must match what your installed
Playwright package expects. Confirm on the target machine with:

```bash
npx playwright install --dry-run
```

If the dry-run shows different build numbers, stop — get the zips that match that
Playwright version instead. (Upgrading Playwright later changes the revision and
requires repeating this process. Pin the Playwright version.)

### Step 3 — Extract into the exact layout Playwright expects

The Chromium zips already contain the correct top-level folder name — do not rename it.
Playwright looks for the executable at `chromium-1234/chrome-linux64/chrome`.
The FFmpeg zip has no inner folder; extracting into `ffmpeg-1011/` is correct as-is.

```bash
CACHE=~/.cache/ms-playwright
mkdir -p $CACHE/chromium-1234 $CACHE/chromium_headless_shell-1234 $CACHE/ffmpeg-1011

unzip ~/pw_transfer/chrome-linux64.zip                 -d $CACHE/chromium-1234/
unzip ~/pw_transfer/chrome-headless-shell-linux64.zip  -d $CACHE/chromium_headless_shell-1234/
unzip ~/pw_transfer/ffmpeg-linux.zip                   -d $CACHE/ffmpeg-1011/
```

Verify the structure:

```bash
ls $CACHE/chromium-1234/                  # should show: chrome-linux64/
ls $CACHE/chromium-1234/chrome-linux64/   # should show: chrome, chrome-wrapper, resources/, ...
```

### Step 4 — Create the validation marker files

Without `INSTALLATION_COMPLETE`, Playwright considers the install broken and tries
to re-download:

```bash
touch $CACHE/chromium-1234/INSTALLATION_COMPLETE
touch $CACHE/chromium_headless_shell-1234/INSTALLATION_COMPLETE
touch $CACHE/ffmpeg-1011/INSTALLATION_COMPLETE
chmod -R u+rwX,go+rX $CACHE
```

(Note: `INSTALLATION_COMPLETE` is the only marker Playwright checks. Extracting on
Linux restores the executable permission bits from the zips — never extract on
Windows and copy folders over.)

### Step 5 — System dependencies

The Chromium binary needs shared libraries. If the machine has apt access:

```bash
sudo apt update
sudo apt install -y libnss3 libnspr4 libatk1.0-0 libatk-bridge2.0-0 \
  libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
  libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libasound2
```

If fully air-gapped, the same `.deb` files must be transferred from another
Debian-based machine of the same architecture.

### Step 6 — Verify

```bash
npx playwright install --dry-run   # should show no pending downloads
```

```python
# Smoke test
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(headless=True)
    page = b.new_page()
    page.goto('data:text/html,<h1>Hello</h1>')
    print(page.content())
    b.close()
```

If launch fails with a missing-library error, Step 5's dependencies are incomplete.

### Gotchas

- **Do not rename the inner folder.** The zip contains `chrome-linux64/` — keep it.
- **Version mismatch:** the folder revision (`chromium-1234`) must match the
  Playwright package version. Pin Playwright in production.
- **Skip Firefox/WebKit** unless needed — same download-and-extract pattern, but
  each has its own revision folder and dependencies.
- **Explicit cache path** if anything is off (add to `~/.bashrc` to persist):

  ```bash
  export PLAYWRIGHT_BROWSERS_PATH=~/.cache/ms-playwright
  ```
