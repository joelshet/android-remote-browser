# android-remote-browser

`arb` drives the real Chrome on an Android phone from the Mac. Pages load, run
and render on the phone with its own network, fonts, viewport and GPU, so what
you capture is what a person holding that phone would see. It's one Python
file with no dependencies beyond `adb`, built for one-off use, scripts and AI
agents alike.

## Requirements

- `adb` on the Mac (`brew install --cask android-platform-tools`) and Python 3.8+.
- A phone with USB debugging on, authorized for this Mac, with Chrome installed.
- No lock screen, or at least a swipe-only one. `arb` wakes the screen and
  brings Chrome to the front before each command, but it can't enter a PIN.
- Keep the phone on a charger. Chrome slows pages down when the screen is off.

Play Store updates Chrome on its own. Each capture records the version it ran
on, and `./arb devices` lists the current one per phone.

With two or more attached, every command needs `-s SERIAL` or
`ANDROID_SERIAL`, since arb only picks a phone by itself when exactly one is
plugged in. Each phone
has its own lock, so two scripts can run side by side, one per phone:

```bash
ANDROID_SERIAL=SERIAL_A ./arb crawl -o runs/a < first-half.txt
ANDROID_SERIAL=SERIAL_B ./arb crawl -o runs/b < second-half.txt
```

## Quick start

```bash
./arb devices
./arb open nytimes.com
./arb snap
./arb shot -o page.png
./arb shot --full -o full.png
```

`./arb -h` lists every command.

## Reading and acting

`open` handles three things before it returns: it waits out a bot check that
clears itself, accepts the cookie banner, and clears an app-install promo.
`--no-dismiss` skips the last two, and `arb dismiss` does them on demand for a
banner that turns up later. It prints the HTTP status, milliseconds to the load
event, the final URL and the title, separated by tabs. It returns once the page has loaded and gone
quiet (two or fewer requests in flight for a second), or 5 seconds after load
on pages whose ads never stop. If one slow resource holds up the load event, it
gives up 10 seconds after the HTML is parsed. `-t` caps the whole wait, 30
seconds by default.

`snap` prints the rendered page as an indented outline: landmarks, headings,
visible text, images with alt text, iframes, and form state. Every control ends
in a ref, including elements that only have a JavaScript click handler
(`clickable`):

```
# Moto G7 - Wikipedia
# https://en.wikipedia.org/wiki/Moto_G7
# viewport 424x759 at y=0 of 3286
header
  link "Wikipedia" url=https://en.wikipedia.org/wiki/Main_Page [e50]
  form
    button "Search" [e111]
main
  heading "Moto G7" level=1
  link "Talk" url=https://en.wikipedia.org/wiki/Talk:Moto_G7 [e63]
```

It's built from Chrome's DOM and layout, not the accessibility tree, because
sites hide visible content from assistive tech. CNN marks its whole page
`aria-hidden`, consent dialog included. Only what Chrome laid out on the page
appears, and controls labelled only for screen readers keep those labels.
Controls with no label at all (bare icon buttons) show without a name.

Refs stay valid across commands until the element leaves the page:

```bash
./arb click e306
./arb type e3064 "Moto G7"
./arb key Enter
./arb scroll down 600
./arb back
```

Taps and scrolls are real touch input, so pages see touch events, and Chrome's
toolbar hides on scroll as it does for a person. Each action waits for any
navigation or loading it sets off. `text`, `html` and `eval` read the current
page. `text` and `html` run in an isolated JavaScript world that the page's own
scripts can't see.

## Bot checks

A check that clears itself (Cloudflare's "Just a moment...") reloads the real
page, so `arb` waits up to 20 seconds for that and carries on. Checks needing a
person (a checkbox, "press and hold", DataDome's 403) are reported, never
answered: `meta.json` gets `"challenge": {"vendor": "...", "cleared": false}`
and `open` says so on stderr. Across a crawl, that field tells you your
challenge rate.

## Cookie banners and app promos

Banners are accepted using [autoconsent](https://github.com/duckduckgo/autoconsent)'s
rules (see `vendor/`), which cover the known consent tools. Autoconsent runs in
an isolated world the page can't see. US sites often roll their own notice
instead (CNN's "Legal Terms and Privacy"), so a fallback taps a plainly worded
accept button, or a close button, inside a consent-worded overlay. Labels are
matched whole, so "Manage cookies" and "Reject all" can't be hit by mistake.

App-install promos are cleared the same way, through controls like "Continue in
browser", "Not now" or a close icon. Anything that would leave the browser
("Open", "Install", "Get the app") is never clicked. Some promos have no way
out at all, like Reddit's bottom sheet; those are reported in `meta.json` under
`app_banner` with `"dismissed": false`.

`meta.json` records what happened: `consent` names the consent tool or the
label clicked and how, and `app_banner` does the same for the promo.

## Screenshots

- `shot` is the visible part of the page, at the phone's full resolution.
- `shot --full` is the whole page. Chrome on Android only paints what's on
  screen, so arb scrolls down the page and stitches the pieces. Fixed and
  sticky elements (headers, cookie bars, anchor ads) show once in the first
  piece and are transparent after that, then restored. Lazy-loaded images
  appear if they load within the 0.3 seconds each piece waits.
- `shot --device` is the phone's screen as it is, including Chrome's address
  bar, the status bar and any system dialog.

Without `-o` the PNG goes to stdout when stdout isn't a terminal.

## Batches

```bash
./arb capture cnn.com -o runs/cnn --shot
./arb crawl -o runs/2026-09-17 < urls.txt
./arb crawl -o runs/raw --no-dismiss < urls.txt   # leave banners in place
```

`capture` writes one directory:

| file | contents |
|---|---|
| `meta.json` | requested and final URL, status, load ms, whether it loaded and went quiet, bot check, consent and app promo outcome, origins contacted, Chrome version, time |
| `snap.txt` | output of `snap` |
| `text.txt` | output of `text` |
| `page.html` | output of `html` |
| `shot.png` | visible page, with `--shot` |
| `full.png` | whole page, with `--full` |

Screenshots are opt-in because they cost the most. A full-page shot of a long
news homepage can take a minute on the G7.

`crawl` captures each URL on stdin into its own subdirectory. It prints status,
load ms, URL and directory per line, or `error` with the reason, and exits 1
if any URL failed. Before each site it clears cookies, the HTTP cache and
storage for every origin the previous site contacted, so each visit is a first
visit. `--keep-state` skips that. URLs that already have a `meta.json` are
skipped, so rerunning a crawl resumes it.

## For agents

Give the agent a shell and this loop: `open` a URL, `snap` to read it, act on
refs with `click`, `type`, `key` and `scroll`, then `snap` again. Use `shot`
when layout matters. All output is plain text, errors go to stderr, and a
failed command exits nonzero.

## One phone, one driver

Each command takes a lock on its phone, and another command waits up to `-t`
plus 30 seconds before failing with "busy". A crawl holds the lock for its
whole run. The lock doesn't cover a sequence of commands, so don't point an
agent at a phone a script is also driving. With several phones attached, pick
one with `-s SERIAL` or `ANDROID_SERIAL`.

## What it leaves behind

- `.locks/` in this folder.
- An `adb forward` from a local port to Chrome's DevTools socket. adb removes it
  when the server restarts, or remove it with `adb forward --remove-all`.

Nothing is installed on the phone.

## License

MIT. `vendor/` holds DuckDuckGo's autoconsent, unmodified, under MPL-2.0 (see
`vendor/README.md`).
