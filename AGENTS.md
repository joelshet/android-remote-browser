# Working in android-remote-browser

`README.md` covers requirements, every command, and what `arb` leaves behind.

- `arb` is one Python file whose only dependency is `adb`. Keep it that way.
- Testing needs a real Android phone on USB with USB debugging authorized. `./arb devices` lists attached phones; with more than one, name one with `-s SERIAL` or `ANDROID_SERIAL`. Each phone takes one command at a time, so don't drive a phone a script is already using.
- `vendor/` holds DuckDuckGo's autoconsent, unmodified, under MPL-2.0. Don't edit those files; `vendor/README.md` says how to update them.
