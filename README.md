# Proxy Subscription Fetcher & Tester

A Python tool that downloads proxy subscriptions, extracts and deduplicates supported proxy configurations, tests them through **Xray**, and saves separate lists for working, Gemini-compatible, and Gemini-plus-Reddit-compatible configurations.

An optional geolocation stage starts Xray again for each eligible configuration, discovers the **public exit IP seen through that proxy**, looks up its country, and writes a separately tagged configuration list.

## Features

- Fetches subscription URLs from `sub_links.txt`.
- Handles plain-text and Base64-encoded subscription responses.
- Extracts proxy links and removes duplicate configurations.
- Tests connectivity through a local Xray SOCKS5 tunnel against:
  - `https://cp.cloudflare.com`
  - `https://www.gstatic.com`
  - `https://captive.apple.com`
- Checks Gemini compatibility after the basic connectivity tests pass.
- Checks Reddit independently and records detailed response diagnostics, including recognized network-security blocks, rate limits, and other HTTP outcomes.
- Saves separate text lists for working, Gemini-compatible, and Gemini-plus-Reddit-compatible configurations.
- Optionally geolocates configurations using the proxy's actual public exit IP, rather than the proxy server's configured hostname or IP.
- Can reuse the newest archive instead of downloading subscriptions again.
- Supports a one-cycle mode for local use and CI workflows.

## Supported configuration formats

The current parser and Xray outbound builder support:

- VMess (`vmess://`)
- VLESS (`vless://`)
- Trojan (`trojan://`)
- Shadowsocks (`ss://`)

The extraction pattern recognizes several additional URI schemes, but those formats are not currently parsed into Xray outbounds. They are therefore skipped and do not enter the test lists.

## Requirements

- Python 3.10 or newer.
- [Xray-core](https://github.com/XTLS/Xray-core) installed and available as `xray` on your `PATH`.
- Python packages:
  - `requests`
  - `PySocks` (required for `socks5h://` proxy support)
- Optional: `maxminddb` and the DB-IP Country Lite database for local country lookups.

Install the Python dependencies:

```bash
python3 -m pip install --upgrade 'requests[socks]' maxminddb
```

`maxminddb` is optional. Without it, or without the database file, the geolocation stage falls back to the configured online geolocation APIs.

## Setup

1. Clone or download this repository.
2. Install Python dependencies and Xray-core.
3. Create `sub_links.txt` in the same directory as the Python script.
4. Add one subscription URL per line.
5. Run the script using one of the commands below.

Example `sub_links.txt`:

```text
https://example.com/subscription-one
https://example.net/subscription-two
# Lines beginning with # are ignored
```

Blank lines and comment lines are ignored. Duplicate subscription URLs are collapsed while preserving their first-seen order.

### Optional local geolocation database

By default, the script looks for a DB-IP Country Lite MMDB file at:

```text
databases/dbip-country-lite.mmdb
```

To use local lookups, obtain the DB-IP Country Lite database under its applicable license and place the extracted `.mmdb` file at that exact path. The `databases/` directory must be relative to the script's current working directory. If the file is missing or cannot be opened, the script logs a message and uses online lookup services instead.

## Running the script

Run one complete fetch-and-test cycle:

```bash
python3 script.py --once
```

Replace `script.py` with the actual filename if it is different.

Run continuously, starting a new cycle every 30 minutes:

```bash
python3 script.py
```

Reuse the newest `*_all_unique.txt` archive and skip subscription fetching:

```bash
python3 script.py --once --last-fetch
```

The `--last-fetch` option can also be used without `--once`; in that case the newest archive is reused on each cycle.

### Runtime behavior

- Without `--once`, the script repeats every 30 minutes (`RUN_EVERY_SECONDS = 30 * 60`).
- With `--once`, it runs a single cycle and exits.
- Xray tests run concurrently using a thread pool (`TEST_WORKERS = 80`).
- Reddit requests have their own concurrency and pacing limits to reduce request bursts.
- Geolocation runs sequentially after the main Xray test workers have completed.

## Configuration

The main settings are near the top of the Python file.

| Setting | Default | Purpose |
|---|---:|---|
| `LINKS_FILE` | `sub_links.txt` beside the script | Subscription source file |
| `FETCH_TIMEOUT_SECONDS` | `5` | Timeout for fetching each subscription |
| `RETRY_COUNT` | `5` | Subscription fetch attempts |
| `RUN_EVERY_SECONDS` | `1800` | Delay between cycles in continuous mode |
| `ARCHIVE_DIR` | `./archive_all` | Main proxy-list output directory |
| `LOG_DIR` | `./logs` | Diagnostic and failure logs |
| `XRAY_BINARY` | `xray` | Xray executable name or path |
| `TEST_WORKERS` | `80` | Concurrent main config tests |
| `ENABLE_GEOLOCATION` | `True` | Master switch for geolocation |
| `GEOLOCATION_CHECK_WORKING` | `False` | Geolocate all configs that pass the base connectivity tests |
| `GEOLOCATION_CHECK_GEMINI` | `False` | Geolocate Gemini-compatible configs |
| `GEOLOCATION_CHECK_GEMINI_REDDIT` | `True` | Geolocate configs compatible with both Gemini and Reddit |
| `GEOLOCATION_DIR` | `./geolocation` | Geolocation output directory |
| `GEOLOCATION_DATABASE_FILE` | `./databases/dbip-country-lite.mmdb` | Optional local country database |
| `GEOLOCATION_REQUEST_DELAY` | `1.5` seconds | Minimum delay between geolocation request starts |
| `GEOLOCATION_MAX_API_ERRORS` | `3` | Maximum recorded API errors per IP before skipping further API lookups |

Edit these constants to change the behavior. The three `GEOLOCATION_CHECK_*` switches select which successful result sets are geolocated. If all are `False`, no geolocation output files are produced even when `ENABLE_GEOLOCATION` is `True`.

### Geolocation method

For each selected configuration, the script:

1. Parses the configuration and builds an Xray outbound.
2. Starts Xray with a temporary local SOCKS5 inbound.
3. Requests a public-IP echo service **through that SOCKS5 tunnel** to identify the proxy's actual public exit IP.
4. Rejects non-global IP addresses.
5. Looks up the exit IP's country using the local DB-IP database first, if available.
6. Falls back to the configured online providers if the local lookup cannot return a country.
7. Rewrites the configuration remark with a country flag, ISO country code, sequence number, and `@Surfboardv2ray` suffix.

The IP-echo requests go through the tested proxy. Country lookup requests to the online geolocation providers are made by the runner itself, not through that proxy.

The geolocation stage uses these configured services:

- `ipwho.is`
- `ipapi.co`
- `ip-api.com`
- `ip.guide`
- `ip.sb`

It tries providers in rotation and caches exit IPs and country results in memory for the current run only. These caches are not persisted between runs.

## Generated files

All output paths are relative to the process's **current working directory**, except `sub_links.txt`, which is resolved relative to the Python script.

### `archive_all/`

Each cycle uses a timestamp prefix in the form `YYYY-MM-DD_HH-MM-SS`.

| File | Contents |
|---|---|
| `*_all_unique.txt` | All extracted, parsed, deduplicated configurations |
| `*_working.txt` | Configurations that pass the three basic connectivity checks |
| `*_working_base64.txt` | Base64-encoded version of the working list |
| `*_gemini_compatible_working.txt` | Configurations that pass the basic checks and Gemini check |
| `*_gemini_compatible_working_base64.txt` | Base64-encoded Gemini-compatible list |
| `*_gemini_reddit_working.txt` | Configurations that pass the basic checks and both Gemini and Reddit checks |

The `*_all_unique.txt` archive is created during a normal fetch cycle. When `--last-fetch` is used, the script reuses the newest matching archive and does not create a new all-unique archive for that cycle.

### `geolocation/`

With the default geolocation switches, one file is produced per cycle:

```text
geolocation/YYYY-MM-DD_HH-MM-SS_gemini_reddit_geolocation.txt
```

The file contains only configurations from the Gemini-plus-Reddit-compatible list for which the script successfully identifies a public exit IP and country and can update the configuration remark.

If enabled, the other result sets produce:

```text
geolocation/YYYY-MM-DD_HH-MM-SS_gemini_geolocation.txt
geolocation/YYYY-MM-DD_HH-MM-SS_geolocation.txt
```

The plain `_geolocation.txt` suffix is used for the `Working` set. Geolocation writes separate files and does not modify the original files in `archive_all/`.

A geolocation output file can be created even if no configurations are successfully resolved; in that case it will be empty. The script logs the number saved and the number skipped.

### `logs/`

The script can write:

- `*_gemini_reddit_checks.txt` — direct-runner Reddit control result and per-config Gemini/Reddit diagnostics.
- `*_detailed_failures.txt` — parsed config details and Xray diagnostics for failed tests.
- `*_failed.txt` — subscription-fetch failures or fatal cycle errors.

Some log types are created only when there are corresponding failures. The Gemini/Reddit diagnostics file is written after the test stage.

## GitHub Actions

The repository can run the script using GitHub Actions. A workflow may:

- run on a schedule or through **Actions → Run workflow**;
- install Python dependencies and Xray-core;
- execute a single cycle with `python script.py --once`;
- upload archive and geolocation directories as downloadable workflow artifacts;
- commit generated output files back to the branch.

For a workflow that pushes generated files, grant the workflow `contents: write` permission. The commit step must stage `geolocation/` as well as `archive_all/`; uploading a directory as an Actions artifact alone does **not** add its files to the Git repository. If `geolocation/` is ignored by `.gitignore`, force-add it in the commit step (`git add -f -A -- geolocation/`).

The workflow's schedule is independent of the script's 30-minute continuous loop. For example, a workflow that invokes `--once` on a six-hour schedule runs one cycle every six hours, not every 30 minutes.

## Important notes

- **Use responsibly.** Only test proxy configurations and subscription sources that you are authorized to use.
- The script sends requests to subscription providers, the connectivity test endpoints, Gemini, Reddit, IP-echo services, and (when needed) geolocation APIs.
- Geolocation identifies the country associated with the observed public exit IP. IP geolocation is approximate and should not be treated as proof of a user's physical location.
- A config may pass the basic connectivity checks but fail Gemini or Reddit; these are separate result categories.
- A generic Reddit HTTP 403 is not automatically classified as a confirmed network-security block. The script looks for known block-page text before assigning that category.
- The geolocation database is optional, and online API providers may be unavailable, rate-limited, or return incomplete data.
- The script does not persist its in-memory IP/country caches between runs.

## Troubleshooting

### `SOCKS proxy support is missing`

Install the SOCKS extra into the same Python environment used to run the script:

```bash
python3 -m pip install 'requests[socks]'
```

### `xray binary not available`

Install Xray-core and verify that this succeeds in the same shell or runner environment:

```bash
xray version
```

### No geolocation file appears

Check that `ENABLE_GEOLOCATION` is `True` and at least one `GEOLOCATION_CHECK_*` switch is enabled. Also check the log output for the selected result-set counts and geolocation errors. With the default settings, only configs that pass both Gemini and Reddit are geolocated.

### Geolocation files are available in Actions but not in the repository

Workflow artifacts and Git commits are separate. Ensure the commit-and-push step stages `geolocation/`, the workflow has `contents: write`, and the push targets the intended branch. Force-add the directory if it is ignored by `.gitignore`.

### No configurations are written to a result list

Check that the subscription response contains supported URI formats, that the configs can be parsed, that Xray is installed, and that the connectivity checks succeed. Review the timestamped files in `logs/` for diagnostics.
