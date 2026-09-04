# Homelab

Three machines at home running a self-hosted stack, an autonomous agent, a
media pipeline, and a growing list of projects I've built to scratch specific
itches — a lecture note-taker, a server health dashboard, an email-in recipe
manager for my family's recipes after they got locked out of a proprietary
site. This repo is the map: what's running, on what, and why.

I started using Claude Code for real work in 2026 and it changed what I
reach for when I have an idea. This repo is honest about that: some of what's
here is mine end to end, some of it I built *with* Claude Code, sitting next
to it the whole way — designing, testing on live hardware, fixing what broke.
Both halves are labeled below. I'm not going to pretend I typed code I
didn't, and I'm not going to undersell the parts I did.

## Hardware

| Machine | Role | Specs |
|---|---|---|
| **bigbox** | main server | Intel i7-7700K, 62 GB RAM, RTX 3050 8 GB, ~16 TB across a ZFS pool + a few extra drives |
| **testserver** ("robbie") | always-on Pi | Raspberry Pi 5, 4-core, 8 GB RAM |
| **office** | GPU laptop | Alienware M17 R4, i7-10870H, RTX 3070 8 GB |

Plus a Home Assistant box (Raspberry Pi) that also acts as a Tailscale subnet
router for the home LAN.

## Network

Everything's on a [Tailscale](https://tailscale.com) mesh — no port-forwarding,
no exposed SSH. Public-facing services (Nextcloud, this recipe app, etc.) go
out through a single [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
from bigbox, so the origin IP is never exposed and there's nothing to punch a
hole in the router for.

## What's running

### Self-hosted stack (bigbox, rootless Podman)

| Service | What it does |
|---|---|
| [Jellyfin](https://jellyfin.org/) | media server |
| [Immich](https://immich.app/) | photo library (self-hosted Google Photos) |
| [Nextcloud](https://nextcloud.com/) | files, notes, calendar |
| [Audiobookshelf](https://www.audiobookshelf.org/) | audiobook server |
| [ComfyUI](https://github.com/comfyanonymous/ComfyUI) | Stable Diffusion workflows |
| [Open WebUI](https://github.com/open-webui/open-webui) | chat UI for local models |
| [Ollama](https://ollama.com/) | local LLM runtime — qwen2.5, qwen3, mistral-nemo, and two vision models, shared across every project below |
| Minecraft (`spencer-smp`) | a server for family and friends |

### Automation

- **openclaw ("Robbie")** — an autonomous agent living on the Pi that handles
  day-to-day requests: file organization, kicking off disc rips, Nextcloud
  notes, that kind of thing.
- **Claude Code**, kept alive on the Pi by a small watchdog service, for
  heavier or longer-running work.

### Media pipeline

- **DVD/Blu-ray ripping** (`dvd-rip.sh`, bigbox) — preflights a disc against
  an expected episode count, matches episode names via the TVMaze API,
  encodes H.265 on the GPU, and only archives the source once every expected
  output file has been verified to exist.
- **RGB lighting control** (office, the Alienware) — `OpenRGB`-driven, applied
  on boot and after resume via a small systemd service.

## Built by me

The stack above — deploying and wiring together Jellyfin/Immich/Nextcloud/
Audiobookshelf/ComfyUI/Ollama behind one Cloudflare tunnel, the DVD ripper,
the RGB controller, and setting up and living with **openclaw/Robbie** as a
daily-driver autonomous agent — is mine. Also:

- **[tattoo](https://github.com/SpencerHalle/tattoo)** — the Bayesian-inference
  plot I got tattooed, hand-derived and hand-coded
- **[phone_scripting](https://github.com/SpencerHalle/phone_scripting)** —
  Android/Tasker automation

## Built with Claude Code

Three real projects, each shipped end to end with Claude Code — from a spoken
idea, through design, onto live hardware, through the bugs that only show up
once real data hits them. Each is its own repo with its actual commit
history, AI co-author trailers included on purpose:

### [lecture-pipeline](https://github.com/SpencerHalle/lecture-pipeline)
Record a lecture on my phone → it's transcribed on a GPU box with Whisper →
an LLM writes supplemental LaTeX notes covering what I didn't catch → commits
straight into my class notes repo. No renaming files, no manual steps.
Debugged along the way: a CDN silently 403ing a default `urllib` user-agent,
two models fighting over 8 GB of VRAM, a reasoning model that would burn its
whole token budget "thinking" and return nothing — and a schedule-based
classifier so a recording never needs a filename to know which class it's from.

### [bigbox-monitor](https://github.com/SpencerHalle/bigbox-monitor)
Server + Minecraft health pushed straight into Home Assistant — CPU, memory,
GPU, disks, ZFS pool health, SMART status per drive, container status, and
who's on the Minecraft server — with a dashboard and phone alerts for the
things that actually matter (a drive failing, a pool degrading, the
monitor itself going dark). No MQTT broker: states post straight to the HA
REST API. The dashboard got built through a hand-rolled WebSocket client,
since HA's dashboard API has no REST equivalent and no library exists for it.

### [recipe-app](https://github.com/SpencerHalle/recipe-app)
My family's recipes got trapped behind a proprietary site that locked them
out. So: email a photo of a recipe card (or a link, or just paste the text)
to one address, and a vision model reads it — handwriting and all — into a
searchable site with a downloadable PDF, with a review queue before anything
goes live. Solved a genuinely fun problem: OAuth for a headless box with no
browser of its own, and two different models that can't share one 8 GB GPU
at the same time.
