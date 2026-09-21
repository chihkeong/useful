# useful

Offline transfer artifacts for air-gapped machines.

## Playwright browsers for Kali Linux (linux64)

Built from `playwright install --dry-run` output — Chromium 151.0.7922.34 (build 1234), matching headless shell, and FFmpeg (build 1011). Attached as assets on the release, not committed to git (GitHub rejects files over 100 MB in a normal push).

Download via the **Releases** page, or on the target machine:

```bash
wget https://github.com/chihkeong-chng/useful/releases/download/playwright-browsers/<asset-name>
```

Install layout expected under `~/.cache/ms-playwright/`:

| Asset | Extract into |
|---|---|
| `chrome-linux64.zip` | `chromium-1234/` |
| `chrome-headless-shell-linux64.zip` | `chromium_headless_shell-1234/` |
| `ffmpeg-linux.zip` | `ffmpeg-1011/` |

After extracting each zip, create `INSTALLATION_COMPLETE` inside each folder so Playwright treats them as installed.
