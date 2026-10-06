# web-netcheck

`web-netcheck` checks whether HTTPS services and their dependent CDN/assets are actually usable from a Linux host, not merely reachable by DNS or TCP.

It was built for cases where filtering/DPI can leave the main site reachable while truncating or breaking downloads from secondary CDN hostnames.

## What it checks

For a profile such as GitHub it can:

- fetch the base page;
- discover HTTPS hostnames referenced by that page;
- combine them with profile-defined static hostnames;
- optionally discover more hostnames from a JSON metadata API;
- verify DNS and HTTPS connectivity for every hostname;
- select random large assets referenced by the page;
- download each selected asset completely;
- compare received bytes with `Content-Length`;
- request the final bytes independently with HTTP Range;
- compare the Range response byte-for-byte with the tail of the full download;
- return a non-zero exit code on mandatory failures.

This is intended to catch failures such as "the first 16 KiB download correctly and the rest is cut off".

## Requirements

Ubuntu 24.04:

```bash
sudo apt update
sudo apt install -y curl dnsutils coreutils jq
```

`jq` is only needed by profiles that use JSON metadata discovery, such as GitHub.

## Install

```bash
sudo install -m 0755 bin/web-netcheck /usr/local/bin/web-netcheck
sudo mkdir -p /etc/web-netcheck
sudo install -m 0644 profiles/github.conf /etc/web-netcheck/github.conf
```

## GitHub check

```bash
web-netcheck github
```

More aggressive asset test:

```bash
web-netcheck github --assets 5 --min-size 1048576
```

IPv6:

```bash
web-netcheck github -6
```

Verbose curl errors:

```bash
web-netcheck github --verbose
```

## Ad-hoc check of another site

```bash
web-netcheck --url https://example.com/ --auto
```

The ad-hoc mode discovers absolute HTTPS URLs from the base HTML and tests the associated hostnames and sufficiently large assets.

For production monitoring, create a profile instead so critical API/download/registry hostnames that are not present on the front page are also covered.

## Add a profile

Copy `profiles/example.conf` to `/etc/web-netcheck/my-service.conf`:

```bash
sudo cp profiles/example.conf /etc/web-netcheck/my-service.conf
sudoedit /etc/web-netcheck/my-service.conf
```

Then run:

```bash
web-netcheck my-service
```

A profile may define:

```bash
BASE_URL="https://example.com/"

STATIC_HOSTS=(
    example.com
    api.example.com
    downloads.example.com
)

DISCOVER_HTML_HOSTS=1
CHECK_ASSETS=1
ASSET_URL_REGEX='^https://cdn\.example\.com/'
ASSET_COUNT=3
MIN_ASSET_SIZE=$((128 * 1024))
RANGE_SIZE=4096
```

## Exit codes

- `0` — mandatory checks passed;
- `1` — endpoint or asset-integrity failure;
- `2` — local configuration, arguments, or dependencies are invalid.

The summary at the end is intentionally easy to parse:

```text
RESULT endpoint_reachability=OK
RESULT metadata_discovery=OK
RESULT asset_integrity=OK
RESULT overall=OK
```

## systemd timer

Example units are included under `systemd/`.

```bash
sudo install -m 0644 systemd/web-netcheck@.service /etc/systemd/system/
sudo install -m 0644 systemd/web-netcheck@.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now web-netcheck@github.timer
```

Logs:

```bash
journalctl -u web-netcheck@github.service
```

## Why static + dynamic discovery

Dynamic discovery alone is insufficient. A front page may use a CDN but never reference endpoints such as container registries, source archives, release downloads, or APIs. Profiles therefore combine:

1. static critical hostnames;
2. hostnames discovered from the current HTML;
3. optional vendor-specific metadata sources.

The GitHub profile uses all three.
