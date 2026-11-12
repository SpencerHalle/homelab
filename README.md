# Homelab

Three machines at home running a self-hosted stack, an autonomous agent, a
media pipeline, and a growing list of projects I've built to scratch specific
itches — a lecture note-taker, a server health dashboard, an email-in recipe
manager for my family's recipes after they got locked out of a proprietary
site, a flashcard app so my sister could ditch Quizlet. This repo is the map: what's running, on what, and why.

I started using Claude Code for real work in 2026 and it changed what I
reach for when I have an idea. This repo is honest about that: some of what's
here is mine end to end, some of it I built *with* Claude Code, sitting next
to it the whole way — designing, testing on live hardware, fixing what broke.
Both halves are labeled below. I'm not going to pretend I typed code I
didn't, and I'm not going to undersell the parts I did.

## Hardware

| Machine | Role | Specs |
|---|---|---|
| **bigbox** | main server | Intel i7-7700K, 62 GB RAM, RTX 3050 8 GB, ~16 TB across a ZFS pool + a few extra drives, two optical drives (BD + DVD) for ripping |
| **testserver** ("robbie") | always-on Pi | Raspberry Pi 5, 4-core, 8 GB RAM |
| **office** | GPU laptop, offsite backup target | Alienware M17 R4, i7-10870H, RTX 3070 8 GB, plus a 4 TB USB drive holding the offsite backups |

Plus a Home Assistant box (Raspberry Pi) that also acts as a Tailscale subnet
router for the home LAN and as an exit node, so it doubles as my VPN home
when I'm on sketchy Wi-Fi. (It's wired next to the router; bigbox lives in
the basement.)

## Network

Everything's on a [Tailscale](https://tailscale.com) mesh — no port-forwarding,
no exposed SSH. Public-facing services (Nextcloud, this recipe app, etc.) go
out from bigbox, either through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
or, for more and more of them now, through a [Caddy](https://caddyserver.com/)
reverse proxy with automatic HTTPS.

bigbox was on Wi-Fi until November 2026. The mesh kept bouncing it between
access points every few minutes and Jellyfin would stutter, so it finally got
a cable. Wired now.

## What's running

### Self-hosted stack (bigbox, rootless Podman)

| Service | What it does |
|---|---|
| [Jellyfin](https://jellyfin.org/) | media server for my disc rips, GPU (NVENC) transcoding + tonemapping |
| [Immich](https://immich.app/) | photo library (self-hosted Google Photos); my whole Google Photos Takeout got imported here, albums and real capture dates intact |
| [Nextcloud](https://nextcloud.com/) | files, notes, calendar; my Google Drive + Keep exports landed here too |
| [Collabora Online](https://www.collaboraonline.com/) | Google-Docs-style editing of documents right inside Nextcloud |
| [RomM](https://romm.app/) | game library, with server-side emulators streamed to the browser (behind RomM's own login) so I can play from anywhere without installing anything |
| [Audiobookshelf](https://www.audiobookshelf.org/) | audiobook server |
| [ComfyUI](https://github.com/comfyanonymous/ComfyUI) | Stable Diffusion workflows |
| [Open WebUI](https://github.com/open-webui/open-webui) | chat UI for local models |
| [Ollama](https://ollama.com/) | local LLM runtime — qwen2.5, qwen3, mistral-nemo, and two vision models, shared across every project below |
| Minecraft (`spencer-smp`) | a server for family and friends |
| FreeBSD VM | plain QEMU/KVM playground VM, just to mess around with something that isn't Linux |

### Automation

- **openclaw ("Robbie")** — an autonomous agent living on the Pi that handles
  day-to-day requests: file organization, kicking off disc rips, Nextcloud
  notes, that kind of thing.
- **Claude Code**, kept alive on the Pi by a small watchdog service, for
  heavier or longer-running work.

### Media pipeline

- **DVD/Blu-ray ripping** (`dvd-rip.sh`, bigbox) — I rip the discs myself
  with MakeMKV and drop them in staging. The script preflights a disc against
  an expected episode count, matches episode names via the TVMaze API,
  encodes H.265 on the GPU (NVENC, CQ 19 — high quality, all audio and
  subtitle tracks kept), and only archives the source once every expected
  output file has been verified to exist. Robbie handles the naming and
  filing into the Jellyfin library. The full-disc rips stay on the ZFS pool
  too.
- **RGB lighting control** (office, the Alienware) — `OpenRGB`-driven, applied
  on boot and after resume via a small systemd service.

## Built by me

The hardware and the network, the original stack — deploying and wiring
together Jellyfin/Immich/Nextcloud/Audiobookshelf/ComfyUI/Ollama behind one
Cloudflare tunnel — the DVD ripper and ripping a whole shelf of discs by hand,
the RGB controller, and setting up and living with **openclaw/Robbie** as a
daily-driver autonomous agent — is mine. Also:

- **[tattoo](https://github.com/SpencerHalle/tattoo)** — the Bayesian-inference
  plot I got tattooed, hand-derived and hand-coded
- **[phone_scripting](https://github.com/SpencerHalle/phone_scripting)** —
  Android/Tasker automation

## Built with Claude Code

Each shipped end to end with Claude Code — from a spoken idea, through design,
onto live hardware, through the bugs that only show up once real data hits
them. The ones with links are their own repos with their actual commit
history, AI co-author trailers included on purpose. The rest live in local
git on bigbox for now.

### [lecture-pipeline](https://github.com/SpencerHalle/lecture-pipeline)
Record a lecture on my phone → it's transcribed on a GPU box with Whisper →
an LLM writes supplemental LaTeX notes covering what I didn't catch → commits
straight into my class notes repo. No renaming files, no manual steps.
Debugged along the way: a CDN silently 403ing a default `urllib` user-agent,
two models fighting over 8 GB of VRAM, a reasoning model that would burn its
whole token budget "thinking" and return nothing — and a schedule-based
classifier so a recording never needs a filename to know which class it's from.

### [recordings](https://github.com/SpencerHalle/recordings)
The front end to the lecture pipeline: a phone-installable voice recorder (PWA)
that saves straight onto bigbox. Hit record during a scheduled class and it
also drops the audio into the exact Nextcloud folder the pipeline watches,
named so the pipeline can work out which class it was — so "record" then
"save" is the whole workflow, and the notes show up half an hour later. Talks
to the Nextcloud container directly across the Podman network to sidestep the
same CDN user-agent block the pipeline ran into.

### [bigbox-monitor](https://github.com/SpencerHalle/bigbox-monitor)
Server + Minecraft health pushed straight into Home Assistant — CPU, memory,
GPU, disks, ZFS pool health, SMART status per drive, container status, and
who's on the Minecraft server, and both backup tiers — with a dashboard and
phone alerts for the things that actually matter (a drive failing, a pool
degrading, a backup going stale, the monitor itself going dark). No MQTT
broker: states post straight to the HA REST API. The dashboard got built through a hand-rolled WebSocket client,
since HA's dashboard API has no REST equivalent and no library exists for it.

### [recipe-app](https://github.com/SpencerHalle/recipe-app)
My family's recipes got trapped behind a proprietary site that locked them
out. So: email a photo of a recipe card (or a link, or just paste the text)
to one address, and a vision model reads it — handwriting and all — into a
searchable site with a downloadable PDF, with a review queue before anything
goes live. Solved a genuinely fun problem: OAuth for a headless box with no
browser of its own, and two different models that can't share one 8 GB GPU
at the same time.

### [bigbox-backup](https://github.com/SpencerHalle/bigbox-backup)
Two tiers, because one drive in the same box isn't really a backup:
1. **Local mirror, nightly** — Nextcloud (in maintenance mode, with a proper
   DB dump), Immich, the recipe app and friends onto a separate drive in
   bigbox. Incremental, usually about a minute.
2. **Offsite, nightly after that** — another machine away from home pulls a
   [restic](https://restic.net/) snapshot of the mirror into an
   **append-only** repo, so even something that owned bigbox couldn't delete
   the history. Pruning only happens on the offsite side. Seeded locally to
   stay under the ISP data cap.

Both tiers report into Home Assistant (status, age, progress, repo size,
free space on the offsite drive), so a backup that quietly stops, stalls, or
gets killed halfway screams at my phone instead of looking fine.

### Self-hosted apps (not on GitHub yet)

- **Flashcards** — an ad-free Quizlet replacement for my sister: paste a
  Quizlet export straight in (with a preview that flags anything it couldn't
  parse), flip cards, a match game, installable as a phone app, data on
  bigbox so it syncs everywhere. Open sign-up with rate limits and a bot
  check, an admin page that shows usage but deliberately *not* anyone's
  cards, an MCP server so Claude can make sets for you, and a banner of joke
  "ads" for my other self-hosted services.
- **Food tracker** — snap a photo, say what you ate, or type it; local
  Whisper on the GPU transcribes, an LLM estimates calories and logs it, and
  every capture can be undone. Per-person accounts, plus a weight log with a
  smoothed trend line, because the trend matters way more than any one
  meal's estimate.
- **Baby log** — one-tap feed / diaper / sleep tracker designed for the 4 a.m.
  path (as few taps as possible), shared between two phones, plus voice notes
  that get transcribed and turned into log edits. The food tracker was
  cloned off this one.
- **Letters** — a test run of a site that collects letters from the public
  and publishes them, with Claude Haiku doing spam / privacy / quality review
  passes before a human editor makes the call.
- **Gospel Library streak keeper** — a headless Android emulator in a
  container that opens the app and reads a chapter twice a day, scrapes the
  streak counter off the screen, and reports it to a Home Assistant
  dashboard that pings me at night if a day got missed. Mostly a joke. A
  long-running joke.

Also done with Claude Code along the way: the Collabora and RomM setups
(including streaming emulators off the GPU from inside a rootless container),
the offsite backup design, getting Jellyfin to actually use the GPU, the
FreeBSD VM, and importing years of Google Takeout into Immich and Nextcloud without losing
dates or albums.
