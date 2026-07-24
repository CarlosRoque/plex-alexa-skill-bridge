# Running standalone (no Docker)

The upstream README deploys via `docker compose`. This fork runs the app directly
under systemd instead. Nothing in the app requires a container — there are no
volumes, no database and no persistent state, and all configuration is environment
variables.

Two reasons to prefer it:

- **Plex on the same host is reachable over loopback.** `PLEX_URL=http://127.0.0.1:32400`
  works directly; a container's `localhost` is not the host's, so the Docker path
  needs Plex exposed on a LAN address.
- One less runtime to install, update and debug for ~50KB of Python.

The tradeoff: upstream publishes a prebuilt image, so `docker compose pull` is the
supported upgrade path. Standalone means `git pull && pip install -r app/requirements.txt`.

## Install

```bash
git clone <your fork> && cd plex-alexa-skill-bridge
python3 -m venv venv
./venv/bin/pip install -r app/requirements.txt

mkdir -p secrets
printf '%s\n' 'YOUR_PLEX_TOKEN' > secrets/plex_token.txt
chmod 600 secrets/plex_token.txt
```

`secrets/` is gitignored. Passing `PLEX_TOKEN` as a *path* rather than the literal
token keeps it out of `systemctl show` output — `_read_secret()` in
`app/src/plex/client.py` reads the file when the value starts with `/`.

## REQUIRED: fix oscrypto before first start

On any host with **OpenSSL 3.x** (Ubuntu 22.04+, Debian 12+, Fedora 36+), the
service will crash-loop on boot:

```
oscrypto.errors.LibraryNotFoundError: Error detecting the version of libcrypto
Worker failed to boot.
```

`oscrypto` 1.3.0 — the newest release on PyPI — cannot parse OpenSSL 3.x version
strings. It arrives transitively: `ask-sdk-webservice-support` → `certvalidator` →
`oscrypto`, and it is what verifies Alexa request signatures, so the app cannot
start without it working.

The fix exists only on git master, which reports the *same* version number — so a
plain `pip install --upgrade` is a silent no-op. Force it:

```bash
./venv/bin/pip install --force-reinstall --no-deps \
  'oscrypto @ git+https://github.com/wbond/oscrypto.git@master'
```

**Re-apply this after any venv rebuild or `pip install -r app/requirements.txt`.**
Do not work around it with `DISABLE_REQUEST_VERIFY=true` on a public endpoint —
that turns off Alexa signature verification entirely.

## systemd

Copy `deploy/plex-alexa-skill.service.example`, substitute the paths and hostname,
install it:

```bash
sudo cp plex-alexa-skill.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now plex-alexa-skill
```

- **`--workers 1` is mandatory.** Queue state is per-process and in-memory;
  multiple workers hand out interleaved tracks.
- Bind `127.0.0.1`, not `0.0.0.0`. The reverse proxy terminates TLS and forwards
  locally; upstream's Dockerfile binds all interfaces only because it is
  containerised.

## Reverse proxy

`apache-vhost.conf` in the repo root is written for httpd/RedHat. On Debian and
Ubuntu, change the logging directives — `logs/...` resolves to `/etc/apache2/logs`,
which does not exist, and **Apache will refuse to start**:

```apache
ErrorLog  ${APACHE_LOG_DIR}/plex_error.log
CustomLog ${APACHE_LOG_DIR}/plex_access.log combined
```

Point the `/library/parts/` and `/library/metadata/` proxy targets at
`http://127.0.0.1:32400` when Plex is on the same host.

If the host runs **fail2ban**, note that giving this vhost its own log files also
takes its traffic out of scope of any jail whose `logpath` is the shared
`access.log`. That is usually what you want — Alexa's servers generate a lot of
requests to `/library/parts/` — but check your own jail config rather than assuming.

## Creating the skill

See `skill/manifest.example.json`. Two things that will otherwise waste your time:

- **Invocation names must be at least two words** unless you own the trademark.
  Upstream's `interaction_model.json` used `plex`, which Amazon rejects outright
  with *"Invocation name must be at least 2 words."*
- **Do not set `distributionMode: PRIVATE`.** It existed for Alexa for Business,
  which is discontinued, and the console then demands you change distribution to
  Public.

Build the interaction model and poll for completion — models using
`AMAZON.Music*` catalog slot types can take **over two minutes**:

```bash
ask smapi get-skill-status --skill-id <id> --resource interactionModel
```

## Invoking it reliably

Amazon Music competes for music-shaped utterances, and on an Alexa+ household that
arbitration is probabilistic — the *same* phrasing can resolve to the skill once
and to Amazon Music the next time. A single successful test proves nothing; repeat
it several times.

The deterministic pattern is to open the skill first and then speak the request, so
the follow-up turn is inside the session where no arbitration happens:

```
"Alexa, open my plex skill"        -> "Welcome to Plex."
"play the artist Fleetwood Mac"     -> plays
```

Appending the word **"skill"** to the launch phrase stops the music domain from
bidding for it.

## Debugging

`journalctl -u plex-alexa-skill -f` is ground truth. In particular, do not trust
`ask smapi simulate-skill`: it has reported `FAILED — "This utterance did not
resolve to any intent in your skill"` for requests the log shows arriving,
resolving to the right intent, querying Plex and returning a valid AudioPlayer
directive. Its NLU is also scoped to the target skill, so it cannot reproduce
arbitration behaviour at all.

Server-side changes (any spoken text lives in `handler.py`) take effect on the next
request after a restart, with no propagation delay. If you hear stale wording, the
process is still running the old code:

```bash
stat -c '%y' app/src/skill/handler.py
systemctl show plex-alexa-skill -p ActiveEnterTimestamp
```

`/status` (set `ENABLE_STATUS_PAGE=true`) reports Plex connectivity, outbound
internet reachability and whether signature verification is on. Turn it back off
when you are done — it is reachable through the proxy and names internal hosts.

## Audio formats

Amazon's AudioPlayer documentation lists only *"AAC/MP4, MP3, PLS, M3U/M3U8, HLS"*.
**FLAC is absent from that list but plays fine** — verified with a 100%-FLAC artist
streaming from `.../file.flac` with no `PlaybackFailed` event. Do not conclude a
format is unsupported merely because the docs omit it. `.wma` and `.wav` remain
untested.
