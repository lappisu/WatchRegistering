# User Monitor

Monitors new user registrations across KoGaMa servers, checks usernames against an embedded flag list, detects probable bot accounts, and forwards results to Discord webhooks.

---

## Overview

User Monitor polls the KoGaMa registration feed, processes each new account in order, and routes it to one of two Discord webhooks depending on whether the username is clean or flagged. All flag terms are compiled directly into the binary — no external files required at runtime beyond the state file and log.

---

## Requirements

- [Rust](https://rustup.rs) 1.75 or later
- Two Discord webhook URLs (one for clean users, one for flagged)
- Network access to the target KoGaMa server(s)

---

## Installation

```bash
git clone $thisrepo
cd scr
cargo build --release
```

The compiled binary is written to `target/release/user-monitor`.

---

## Configuration

Copy `.env.example` to `.env` and fill in the required values.

```bash
cp .env.example .env
```

### Variables

| Variable          | Required | Default            | Description                                              |
|-------------------|----------|--------------------|----------------------------------------------------------|
| `WEBHOOK_VALID`   | Yes      |                    | Discord webhook URL for clean users                      |
| `WEBHOOK_FLAGGED` | Yes      |                    | Discord webhook URL for flagged and bot users            |
| `SERVERS`         | Yes      |                    | Comma-separated server base URLs                         |
| `START_ID`        | Yes      |                    | User ID to begin from on first run                       |
| `STATE_FILE`      | No       | `user_state.json`  | Path to the progress file                                |
| `POLL_INTERVAL`   | No       | `15`               | Seconds between poll cycles                              |
| `WEBHOOK_DELAY`   | No       | `1.8`              | Seconds between each webhook send                        |
| `API_TIMEOUT`     | No       | `10`               | HTTP timeout in seconds                                  |
| `LOG_FILE`        | No       | `flagged.log`      | Path to the flagged user log                             |
| `LOG_LEVEL`       | No       | `info`             | `trace`, `debug`, `info`, `warn`, or `error`             |

### Example `.env`

```env
WEBHOOK_VALID=https://discord.com/api/webhooks/123456789/your-token
WEBHOOK_FLAGGED=https://discord.com/api/webhooks/987654321/your-token
SERVERS=https://www.kogama.com,https://friends.kogama.com,https://kogama.com.br
START_ID=670496074
```

---

## Usage

```bash
# Development
cargo run

# Production
./target/release/user-monitor
```

Press `Ctrl-C` for a clean shutdown. State is saved to `STATE_FILE` before exit.

---

## How It Works

Each poll cycle:

1. Fetches `/user/?page=1&count=400` from each configured server
2. Paginates further pages if new users exceed the first page
3. Filters out any user IDs already processed
4. Sorts remaining users oldest-first and processes each in order
5. Checks the username against the embedded flag list
6. Runs bot detection against the last 30 usernames seen on that server
7. Sends the result to the appropriate webhook
8. Saves progress to the state file after each user

---

## State File

Progress is stored per-server in a JSON file:

```json
{
  "https://www.kogama.com": 670496074,
  "https://friends.kogama.com": 12345678
}
```

`START_ID` is only used on first run when the state file has no entry for a server. Once an entry exists, the state file always takes priority and `START_ID` is ignored.

To resume from a specific point, edit the state file directly and restart.

---

## Verdicts

Each user receives one of four verdicts:

| Verdict        | Colour  | Meaning                                        |
|----------------|---------|------------------------------------------------|
| `Clean`        | Green   | No flags, no bot signals                       |
| `Flagged`      | Red     | Username matched one or more flag terms        |
| `Bot`          | Indigo  | Bot detection triggered, no flag terms matched |
| `Flagged + Bot`| Amber   | Both flag match and bot detection triggered    |

Clean users are sent to `WEBHOOK_VALID`. All other verdicts are sent to `WEBHOOK_FLAGGED` and written to `LOG_FILE`.

---

## Flag List

Flag terms are embedded at compile time from `flags.txt`. The file is organised by language and covers English, Russian, Polish, French, Portuguese, Spanish, German, Dutch, Croatian/Serbian/Bosnian, Slovene, and Italian.

Terms are matched after:
- Unicode normalisation and diacritic stripping
- Cyrillic and Greek lookalike mapping to ASCII
- Leet-speak substitution (`4` -> `a`, `3` -> `e`, `0` -> `o`, etc.)
- Non-alphanumeric stripping (catches `f.a.g`, `n-i-g-g-e-r`, etc.)

To update the flag list, edit `flags.txt` and recompile.

---

## Bot Detection

Each server maintains an independent sliding window of the last 30 normalised usernames. A username is flagged as a probable bot if any of the following apply:

- **Sequential numbering** -- the username shares a base with a recent entry and the numeric suffix differs by 2 or less (e.g. `player123` after `player121`)
- **High similarity** -- Levenshtein similarity of 85% or more against any username in the window
- **Repeated base** -- the same alphanumeric base appears 3 or more times in the window

Detectors are kept separate per server to avoid cross-server false positives.

---

## Log Format

Flagged users are appended to `LOG_FILE` in the following format:

```
2026-03-06T11:12:00Z | https://www.kogama.com | xXxNazixXx | id:670494082 | verdict:Flagged | flags:[nazi] | bot:no
```

---

## Multi-Server Support

Set `SERVERS` to a comma-separated list of base URLs:

```env
SERVERS=https://www.kogama.com,https://friends.kogama.com,https://kogama.com.br
```

Each server is polled independently with its own state and bot detection window.

---

## Running as a Service

### systemd

Create `/etc/systemd/system/user-monitor.service`:

```ini
[Unit]
Description=KoGaMa User Monitor
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/opt/user-monitor
EnvironmentFile=/opt/user-monitor/.env
ExecStart=/opt/user-monitor/user-monitor
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now user-monitor
sudo systemctl status user-monitor
```

### Docker

```dockerfile
FROM debian:bookworm-slim
WORKDIR /app
COPY target/release/user-monitor .
COPY .env .
CMD ["./user-monitor"]
```

```bash
docker build -t user-monitor .
docker run -d --name user-monitor \
  -v "$(pwd)/user_state.json:/app/user_state.json" \
  -v "$(pwd)/flagged.log:/app/flagged.log" \
  user-monitor
```

---

## Troubleshooting

**No users appearing in webhook**
Check `LOG_LEVEL=debug` output. Confirm `START_ID` is not higher than the current latest registration ID on the server.

**State file being ignored**
Delete `user_state.json` and set `START_ID` to the correct value, then restart.

**False positives in bot detection**
Increase the similarity threshold in `src/main.rs` (`0.85`) or widen the sequential gap tolerance (`<= 2`), then recompile.

**Panic on unicode username**
Update to the latest version -- an earlier build had a byte-boundary panic on multibyte usernames that has since been fixed.

---

## Disclaimer

This tool is provided for legitimate moderation purposes. Users are responsible for ensuring their usage complies with applicable laws, platform terms of service, and data protection requirements. The authors accept no liability for misuse.
