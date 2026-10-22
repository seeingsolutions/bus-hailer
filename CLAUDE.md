# Bus Hailer: project notes

A VoiceOver-first web app that drives a Bluetooth LED sign so a blind or low-vision person can show a bus number to an approaching driver. Built for Seeing Solutions (Alexander). Live at https://seeingsolutions.github.io/bus-hailer/ (GitHub Pages from `main`, root). The old address seeing.solutions/bus-hailer-v1_0_0.html redirects there.

## How to work on it

- The whole app is `index.html`, one self-contained file. Edit it directly.
- Bump the version in the `Version x.y.z` line at the bottom of Setup with every change, and put the version in the commit message. Alexander checks that line to confirm Bluefy has the new copy.
- Deploy by pushing to `main`. GitHub Pages takes a minute or two, and Bluefy may cache for up to 10 minutes.
- Alexander tests on the real sign and reports back, often with photos or a pasted connection log (Setup, then Copy log). Hardware facts below are marked tested only when he confirmed them.
- Headless check: Playwright with Chromium at `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`. Google Fonts don't load there, so text widths differ from the phone; don't trust layout widths from headless tests.
- Preferences: version numbers in any downloadable filenames; send only changed files.

## The sign

- CoolLEDUX panel, Bluetooth name `CoolLEDUX` (ID 01A7), 32 wide by 16 tall, full colour, 9 program slots. Official app: CoolLED1248.
- Phone side: iPhone with VoiceOver, using the Bluefy browser (Safari has no Web Bluetooth).
- Power: needs a USB-A to C cable. A C-to-C cable gives no power, probably because the CC resistors are missing.
- The start-up ID display was turned off in the CoolLED1248 app. That setting lives in the sign.

## Protocol (sources: robot-pizza/led wire_format, zackmcmurrin-dev/coolledux-linux-uploader)

- Service `0000fff0-0000-1000-8000-00805f9b34fb`, characteristic `0000fff1-...` for write without response and notify. **Bluefy rejects the short numeric UUID form, so use the full strings.**
- Frame: `01`, then a 2-byte big-endian length, the payload, and `03`. Bytes 01 to 03 inside the frame are escaped as `02` followed by the byte XOR 04. Writes go in chunks of 180 bytes with about 12 ms between them.
- Login: `0D 55` then each PIN digit XOR 0x55, then an XOR checksum. The default PIN 000000 gives `0d 55 55 55 55 55 55 55 00`, and the reply is `0d 00`.
- Info: `1F`. Reply byte 8 is the slot count (9).
- Brightness `04 nn`. Power `05 00/01`. Flip `0C 00/01`. The flip command is untested; a double tap on the cable button also flips.
- Upload: start packet `02 crc32 len index count show`, then data chunks of 1024 bytes, each acknowledged by `03 00 idx 00`. The program is LZSS-encoded; this app sends literal-only LZSS.
- Program wrapper: 7 zero bytes, a random nonce, the item count, `00`, then the items.
- Text program: a colour item (0x06) followed by a text item (0x01). Glyphs are column-major with the top row in the MSB, 2 bytes per column.
- **Emoji glyph (item type 1) carries its own RGB444 pixels, column-major, panel height. Tested: the stop sign renders correctly in steady and scrolling text.**
- Picture program: type `03 01`, using the animation layout from zack's uploader. Single and multi-frame both work. Frame delays are u16 milliseconds.

## Behaviour found by testing (important)

- **`show` (last start byte) = how many times a preset plays before the sign moves on.**
  - Moving content (scrolling, word-by-word animation, flashing) must use 0, which repeats forever.
  - Steady text with 0 redraws every couple of seconds, so steady presets use the set size (9).
- **Writing slot 0 starts a fresh set and wipes the others.** So the whole set is always written in order, slots 0 to 8, on every save and on every phone message.
- **Unused slots show rainbow demo content, and the cable button steps onto them.** So all 9 slots are always filled, with spare ones showing a dim grey `EMPTY`. EMPTY was chosen because it can't be mistaken for a route number. Tested in v1.19.0: EMPTY looks good and the grey is fine.
- Tested in v1.19.0: a phone send (all 9 slots) takes about 5 seconds.
- The sign resumes on the last shown preset after a power cycle. Cable button presses are not reported over Bluetooth.
- A one-frame animation plus a blank gap just blinks. Word-by-word must have at least 2 frames.
- Narrowed bold lettering destroys diagonal letters (A and X were unreadable). Words with A, K, M, N, V, W, X, Y, Z, 2, 4 or 7 use the narrow LED alphabet instead. Tested in v1.19.0: the X in ALEX is clear in the narrow alphabet, so it stays 4 wide.

## Current design (v1.19.0)

- Preset 1 holds the last phone message, presets 2 to 9 hold up to 8 favourites, and spare presets show EMPTY.
- Wording options: number only, number with stop sign (12 by 12 red octagon with a white bar, sent as an emoji glyph), number with hand, or number then STOP.
- Lettering order for a steady screen:
  1. Bold Atkinson Hyperlegible at large, then medium.
  2. The tall 5-wide LED alphabet at 14 rows.
  3. Narrowed bold, tallest first, only for words without diagonal letters.
  4. The narrow 4-wide LED alphabet.
  5. The small sizes.
- Text too long for one screen goes word by word: each word held steady, same size for every word, then a short blank before it repeats. It scrolls only if a single word can't fit.
- Scroll speed slider 1 to 10 maps to wire values 180 to 250. The default is 7.

## Open items

- Send speed (about 5 s). Untested ideas: (a) check whether the new number appears on the sign after slot 0 finishes or only at the end; (b) if writing a slot other than 0 does not wipe the set, the phone message could live in one slot that is rewritten alone. Skipping unchanged slots can't work while the phone message is in slot 0, because writing slot 0 wipes the rest.
- Simplification pass. The main screen becomes favourites, one bus-number box and one big Show button, with Automatic deciding the layout. Everything else moves into Setup.
- "Reset sign for a customer" step: writes a clean set before handover. A new sign arrives full of demos.
- Later:
  - Native iPhone app for customers: Capacitor or Expo, built in the cloud because Alexander has no Mac. Needs an Apple Developer account.
  - ESP32 controller in the handle, with tactile buttons and spoken or vibrating feedback of which preset is showing.
  - Case design: a flat paddle with an 18650 battery in a cylindrical handle and a smoked polycarbonate front. Don't roll the panel repeatedly.
