# Attention Ledger

A private, self-hosted record of where my time and attention actually go across my Windows laptop and iPhone.

Small collectors on each device write to storage I own: a database on my laptop and a folder in my own iCloud Drive. The only reports it produces are exported to my laptop for me to read. Nothing is sent to a server, an account or any third party.

**This repository is the public design.** It explains how the system works and how it keeps the data private. The source code, and every minute it has recorded, stay private.

**[Read the full write-up, with interactive examples →](https://sriharish252.github.io/attention-ledger/)**

![A sample day in the preview: time on task, longest focus, switches per hour, distractions and phone checks, above a timeline with one lane for the laptop and one for the iPhone, coloured by focus, communication, distraction and other.](images/day-timeline.png)

<sub>All screenshots use invented sample data.</sub>

## Why

Screen-time apps tell you how long each app was open. I wanted to know what I was actually doing, when I drifted, and what pulled me away, across both screens at once, without handing a minute-by-minute record of my life to anyone.

## Principles

- **Private by design.** No keystrokes, screenshots, audio, message content, full web addresses or raw window titles are ever recorded. Only app names, website domains, and cleaned titles for apps I explicitly allow.
- **Only storage I own.** Data stays on my own devices and in my own iCloud Drive folder. No telemetry, no accounts, no servers, no third-party analytics.
- **I stay in control.** Any device can pause itself or every device. Any app or site can be blocked, and is then logged only as `Private` so the day's total still adds up.
- **Independent parts.** Each collector was built, tested and used on its own device for at least a day before any data was combined.

## How it works

![How the data moves: collectors on the laptop, browsers and iPhone write to a database on the laptop and a private iCloud Drive folder, which feed one merged timeline and exports for me. All of it sits inside a boundary labelled "my devices and my own iCloud Drive". Outside it: no servers, no accounts, no telemetry, no third-party analytics.](images/data-flow.png)

```mermaid
flowchart LR
  subgraph mine["My devices and my own iCloud Drive"]
    subgraph laptop["Windows laptop"]
      agent["Laptop agent<br/>foreground app · idle · lock"]
      ext["Browser extension<br/>active tab's domain only"]
      db[("Local database<br/>raw events kept 30 days")]
      ext -- "loopback address only" --> agent
      agent --> db
    end
    subgraph phone["iPhone"]
      shortcuts["Shortcuts automations<br/>app opened · app closed"]
    end
    icloud[("Private iCloud Drive folder<br/>day files · block list · pause switch")]
    shortcuts --> icloud
    icloud -. "pause switch" .- agent
    db --> analysis["Analysis<br/>one merged timeline"]
    icloud -- "iCloud sync" --> analysis
    analysis --> exports["Daily report + distractions spreadsheet<br/>in a folder on my laptop"]
  end
```

The only network hop is iCloud syncing my own folder between my own phone and laptop. The browser extension talks to the laptop agent over the laptop's loopback address, which never leaves the machine.

### The parts

| Part | Reads | Never reads | Writes to |
|---|---|---|---|
| **Laptop agent** (Windows tray app) | The foreground window's app, time since the last input (not the input), screen lock, whether music is playing | Keys, clicks, screen contents; window titles unless the app is on an allowlist, and then only a cleaned piece | A database in my Windows profile, exported as one file per day |
| **Browser extension** (Chrome, Edge and other Chromium browsers) | The active tab's address, reduced to its domain the moment it is seen; which tabs are playing sound | Page contents, paths, search terms; browser pages, local files and local addresses are skipped | The browser's local storage, then the laptop agent. Browser sync is never used |
| **iPhone automations** (Shortcuts, with two small Scriptable scripts) | That an app opened or closed; Focus mode, charger and step-count signals | Anything inside an app | One file per day in my iCloud Drive, with a local fallback on the phone |
| **Analysis** (Python, nothing to install) | The day files from both devices | Nothing else, and it makes no network calls | A report and a spreadsheet file on my laptop |

## Where the data lives

| Where | What | Kept | Who can see it |
|---|---|---|---|
| Database on my laptop | Laptop and browser events | 30 days | Me, on my Windows account |
| The browser's own storage | A queue for the agent, and its own copy | 7 days | Me, in that browser profile |
| My iCloud Drive folder | The phone's day files, the block list, the pause switch | Until I delete them | Me, on my Apple account |
| A logs folder on my laptop | Day files, reports, the distractions spreadsheet, my categories | Until I delete them | Me. Ignored by git, so never committed |

### What gets stored, and what doesn't

Window titles and web addresses are reduced before anything is written. A browser tab keeps only its registrable domain. A window title is kept only for apps on an allowlist, and then only the part before the first ` - `, with emails, web addresses, file paths, phone numbers and long runs of digits removed.

![The privacy filter example: a YouTube address goes in, the path and query are dropped, and the stored line holds only the domain youtube.com.](images/privacy-filter.png)

Never recorded:

- Keystrokes or mouse input
- Screenshots or screen contents
- Audio, messages, or what was playing
- Full web addresses, paths or search terms
- Raw window titles
- Names of blocked apps and sites: they appear as `Private`, with no title or domain

## The event format

Every collector writes the same format: [JSON Lines](https://jsonlines.org/), one event per line, one file per device per day. The format allows a fixed set of fields and **nothing else**, so a collector can't start recording something new, like a full address, without the format, its validator and its examples changing together.

```json
{"schema_version":1,"ts":"2026-11-09T09:25:30-05:00","device":"laptop","source":"browser-extension","event":"span","app":"msedge.exe","domain":"github.com","duration_s":240}
{"schema_version":1,"ts":"2026-11-09T09:31:12-05:00","device":"phone","source":"ios-shortcuts","event":"open","app":"Instagram"}
{"schema_version":1,"ts":"2026-11-09T09:41:00-05:00","device":"laptop","source":"desktop-agent","event":"span","app":"Private","private":true,"duration_s":95}
```

| Field | Meaning |
|---|---|
| `schema_version` | Always `1` for now |
| `ts` | ISO-8601 time **with** its UTC offset, so laptop and phone logs line up |
| `device` | `laptop` or `phone` |
| `source` | `desktop-agent`, `browser-extension`, `ios-shortcuts` or `ios-app` |
| `event` | `span`, `media`, `open`, `close`, `lock`, `unlock`, `idle_start`, `idle_end`, `pause`, `resume`, `label` or `context` |
| `app` | App or process name; `Private` for blocked apps |
| `domain` | Registrable domain only, lowercase. Never a path, query or full address |
| `title` | A cleaned window title, at most 200 characters, only for allowlisted apps |
| `duration_s` | Length of a span, 0 to 86,400 seconds |
| `private` | `true` when the app or site was blocked; then `app` must be `Private` with no `title` or `domain` |
| `paused`, `until`, `scope` | Tracking state, when a pause ends, and whether it covered one device or all |
| `label`, `value` | A short label or background signal, like `focus:work` or a step count |

## How the report decides

![The distractions table for a sample day: the time, whether it was a distraction, phone check or detour, the task I left, what I did instead, on which device, how long, and what I came back to.](images/distractions.png)

- **On a task:** the app or site in front is in a focus category (work and career by default), or was less than 2 minutes ago.
- **Distraction:** leaving a task for something in the distraction category, on either device, for 10 seconds or more.
- **Phone check:** picking up the phone during a task for anything else, like a message or the clock.
- **Detour:** a minute or more on something else on the laptop. Shorter detours, like a quick search, aren't counted.
- **Passing through:** anything in front for under 3 seconds is folded into what came before, so switching past windows isn't counted as switching.
- **Phone sessions:** iOS has no "phone locked" signal, so an app with no close event ends at the first of: the next app, activity on the laptop, a Sleep Focus or charger change, or a cap (45 minutes for video, calls and maps; 20 for everything else).

Categories are mine to edit in a settings file kept with my logs. Moving an app between categories regroups every past day the next time the report runs.

![A sample day's time grouped by app or site, coloured by whether each counts as focus, communication or distraction.](images/grouped.png)

![Focus minutes per hour across a sample week, Monday to Friday, 9am to 5:30pm, in shades of one blue.](images/week-heatmap.png)

## Pausing and blocking

Any device can pause itself, or every device, for 15 minutes, an hour or until resumed. While paused, each device writes exactly one `pause` line and one `resume` line, and nothing in between. A pause for all devices travels through a small switch file in the shared iCloud folder, so no server is involved:

```mermaid
sequenceDiagram
  participant P as iPhone
  participant C as My iCloud Drive folder
  participant A as Laptop agent
  participant B as Browsers
  P->>P: Toggle Tracking: all devices, 1 hour
  Note over P: one pause line, then nothing
  P->>C: flip the pause switch
  C-->>A: iCloud sync, usually within a minute
  A->>A: notices within 5 seconds, pauses
  A->>B: Paused: all devices
  Note over A,B: nothing logged until resume
```

A pause started on the laptop or in a browser reaches the phone the same way: the next time an automation runs, it sees the switch and logs nothing.

![The pause example: started on the iPhone for all devices, the laptop agent, browsers, iCloud switch and iPhone all show paused, with each step listed.](images/pause.png)

Blocking works from either side. Blocked apps and sites are logged as `Private`, so total time still adds up without the name ever being written.

## Hard parts

- **iOS won't share app usage.** Screen Time data can only be shown inside Apple's sandboxed view and can't be exported. Personal Shortcuts automations ("when Instagram opens, run this") are the one route into a file I own.
- **There is no "phone locked" signal.** Phone sessions are ended by the rules above instead.
- **Two collectors see the same screen.** The agent sees that a browser is in front; the extension knows which site. The timeline takes the site from the extension.
- **A pause has to reach every device without a server.** The shared iCloud folder doubles as the switch.
- **Making a leak impossible, not just unlikely.** The strict format means a new kind of data needs a deliberate format change.

## Testing

Each part has its own automated tests, about 250 in all, run on every change on Windows and Ubuntu, plus a manual checklist on the real device. A part counted as done only once its tests passed and it had run on the real device for at least a day.

## Status

In daily use on my laptop, browsers and iPhone since November 2026. An optional iPhone companion app for motion data and quick labels may come later.

---

Built by [Sri Harish Jayaram](https://sriharish252.github.io/). Reach me on [LinkedIn](https://www.linkedin.com/in/sri-harish).

© 2026 Sri Harish Jayaram. All rights reserved. This repository describes a design; it contains no source code.
