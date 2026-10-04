> [!IMPORTANT]
> **Omanotch is part of [OmacVM](https://github.com/gillesgoetsch/omacvm) now.**
> It lives in [`src/omanotch`](https://github.com/gillesgoetsch/omacvm/tree/main/src/omanotch),
> with its full history. New work, issues and pull requests go there; this repo
> stays as it is and gets no more updates.
>
> With OmacVM you need nothing else: on a MacBook with a notch it sets Omanotch
> up on the Mac and in the VM (`omacvm enable omanotch` on an existing VM).
> Already installed from this repo? `omacvm update` switches you over, and
> `~/omanotch` can go.

<h1 align="center">Omanotch</h1>

<h3 align="center">Enabling the MacBook notch in Omarchy VMs</h3>

<p align="center">The real Omarchy bar, right where Parallels (or UTM) leaves a black hole.</p>

<p align="center">
  <b>Part of the <a href="https://github.com/gillesgoetsch/omacvm">OmacVM</a> experience</b>: OmacVM builds the whole Omarchy VM on your Mac in one command and sets Omanotch up with it, along with trackpad gestures, macOS-like scrolling and the Mac's Wi-Fi, audio and keys in Omarchy.<br>
  <code>curl -fsSL https://raw.githubusercontent.com/gillesgoetsch/omacvm/main/install.sh | bash</code>
</p>

<p align="center">
  <img src="docs/hero.svg" alt="Animated diagram: the VM leaves the notch strip black; inside the VM Omarchy renders its bar on an invisible monitor; Omanotch streams the changed pixels into the strip, the windows grow to full height, and a click on the clock travels back and opens the calendar right below the notch." width="100%">
</p>

You run [Omarchy](https://omarchy.org) full screen in a Parallels or UTM VM on a
MacBook with a notch. It is fast, it is beautiful, and it has a black bar
across the top that nobody asked for. **Omanotch** puts Omarchy's **real** bar
into that black strip — not a look-alike, the actual Quickshell bar, pixels and
all — and gives the space the bar used to take back to your windows.

> [!NOTE]
> **Got an M1 or M2 Mac?** Run Omarchy natively on [Asahi Linux](https://asahilinux.org)
> instead — no VM, full hardware, and it uses the notch area itself. This project
> is for **M3 and M4** Macs, which Asahi does not support yet, so a VM is the way
> to run Omarchy there. (It works on M1 and M2 too.)

<p align="center">
  <img src="docs/before-after.svg" alt="Before: a black strip above the VM plus Omarchy's bar inside it. After: the bar sits beside the notch and the windows use the whole screen below." width="100%">
</p>

## The black strip (and why nobody can fix it)

On a notched MacBook, Parallels and UTM put their full-screen window *below* the
camera housing. The strip beside the notch — the menu bar's height, 43 points on
a 16-inch MacBook Pro at "More Space", a little less on smaller models — stays
black, and Omarchy then draws its own 26-point bar underneath. About 69 points
of your screen, gone.

Can't we just make the VM use that area?

- **Parallels:** there is no setting, documented or hidden. Parallels staff
  [said so on their forum](https://forum.parallels.com/threads/2021-16-macbook-fullscreen-over-notch.355917/):
  drawing into the notch area would go against Apple's guidelines.
- **UTM:** 5.0.6 can draw into the notch area, but only on macOS 27.
- **macOS:** only the app that owns a window may place it next to the notch.
  Moving Parallels' window there from the outside is simply refused (tried it,
  also with the menu bar on auto-hide).
- **Code injection** into the VM app would work in theory, but needs System
  Integrity Protection turned off. No thanks.

So the VM cannot go up there. **But a tiny Mac app of our own can** — and it can
show whatever the VM would have shown.

## How the trick works

1. **An invisible monitor.** Inside the VM, Hyprland gets an extra, headless
   output called `NOTCH`: exactly as wide as your screen, exactly as tall as the
   bar. Nobody ever sees it. It sits *on top of* the real display's top edge,
   which sounds wrong but is the whole point — Parallels' mouse and UTM's USB
   tablet are mapped over the bounding box of all monitors, so an extra monitor
   anywhere else would shift every click.
2. **The bar, twice.** Omarchy's bar is cloned with Omarchy's own
   `omarchy plugin clone` and patched: one copy renders on `NOTCH` (the pixels
   you will see), the copy on the real display shrinks to 1 px and hides just
   off screen. It is not gone, though — your clicks are pressed on that hidden
   copy, so Omarchy opens its panels (clock, audio, network, …) on the visible
   display, right below the notch. The wallpaper is patched the same way: it is
   laid out once across the strip and the display, so with the bar hidden
   (Super+Shift+Space) the image runs straight through the notch strip.
3. **Streaming only what changes.** `notchcast`, a small C program in the VM,
   captures `NOTCH` with Wayland's `ext-image-copy-capture`. A capture only
   completes when Hyprland actually repaints, so an idle bar costs zero CPU. It
   sends just the rectangle that changed, LZ4-compressed — usually 0.5–3 KB —
   over the VM's private network. It finds the Mac by itself.
4. **A panel above everything.** *Omanotch.app* draws the frames in a
   borderless panel at window level 27 — above the menu bar (24) and above an
   invisible window Parallels and UTM keep over the strip (26). It never takes focus,
   so your keyboard stays with the VM. It lives on the VM's full-screen Space
   and slides with it when you swipe.
5. **Cursor juggling.** Over the strip, the Mac shows the *guest's* cursor
   images (sent over from the VM) while the VM hides its own; over the VM, the
   macOS cursor is hidden for real. One cursor at a time.
6. **Fail-safe.** The Mac app says "keep the bar parked" every second. If you
   leave full screen or the Mac app quits or crashes, the VM brings its bar
   back at once; if even the program in the VM dies, within 15 seconds.

## What it handles

<p align="center">
  <img src="docs/states.svg" alt="Animated loop: Omanotch bar beside the notch, a notification right under it, a theme switch, the bar hidden with the wallpaper running through the strip, full-screen video with a black strip, the lock screen with a black strip, 16/14/13-inch MacBooks with the notch gap staying aligned, and swiping Spaces with the strip travelling along" width="100%">
</p>

Notifications never cut into the strip, theme switches don't make it jump,
hiding the bar lets the wallpaper run through behind the notch, full-screen
video and the lock screen turn it black, and it follows you across Spaces and
MacBook sizes.

## Any notched MacBook

Nothing is tied to one model. The Mac app measures everything on the spot:
where the camera housing is, how tall the strip is, and how many guest pixels
land on one Mac point. 13- and 15-inch MacBook Air, 14- and 16-inch MacBook
Pro, any "Larger Text" to "More Space" setting — the invisible monitor, the
notch gap in the bar and every click follow along, also when you change the
resolution while the VM is running. If the VM's resolution does not match the
Mac point for point, the strip is scaled to fit instead of cut off.

## Requirements

- A MacBook with a notch, macOS 14 or later, Xcode Command Line Tools
- Omarchy (Arch Linux ARM, Hyprland 0.56 or newer) full screen on the built-in
  display, in
  - **Parallels Desktop**, with the shared network (the default), or
  - **UTM** 5, with the shared network (the default) and **automatic mouse
    capture off**: UTM → Settings → Input → uncheck both *Capture input
    automatically…* options. A captured mouse can never reach the strip. For a
    pixel-sharp strip turn on the display's *Retina Mode* (off by default) and
    keep *Resize display to window size automatically* on (the default).
- In the VM: `gcc`, `wayland`, `wayland-protocols`, `lz4`, `python3` — all
  already there on Omarchy

<sub>Parallels or UTM? Both work. Parallels is faster and drives external
monitors too: [some numbers](docs/why-parallels.md).</sub>

## Install

In the VM, as your normal user, from a checkout of this repository:

```bash
./guest/install.sh
```

On the Mac:

```bash
./mac/install.sh
```

That builds `~/Applications/Omanotch.app` and starts it at login
(log: `~/Library/Logs/omanotch.log`). Put the VM in full screen on the built-in
display and the bar moves into the strip.

## Uninstall

```bash
./guest/uninstall.sh      # in the VM (add --remove-bar-clone to drop the bar clone too)
./mac/uninstall.sh        # on the Mac
```

## Configuration

Mac app — `defaults write ch.gillesgoetsch.omanotch <key> <value>`, then
`launchctl kickstart -k gui/$(id -u)/ch.gillesgoetsch.omanotch`:

| Key | Default | |
|---|---|---|
| `vmOwners` | `Parallels Desktop`, `UTM` | apps whose full-screen window is the VM (`-array …`) |
| `vmInterfacePrefixes` | `bridge`, `vnic` | VM network interfaces the Mac listens on … |
| `vmSubnets` | `192.168.64.0/24`, `10.211.55.0/24`, `10.37.129.0/24` | … if their network is one of these (UTM, Parallels shared, Parallels host-only); guests are accepted only from that network |
| `listenHost` | *(automatic)* | listen on this one IPv4 address instead |
| `port` | `47811` | |

VM — `systemctl --user edit notchcast`, `Environment=…`:

| Variable | Default | |
|---|---|---|
| `NOTCHBAR_HOST` | *(automatic)* | the Mac's address (list allowed); by default the default gateway (UTM) and `.2` of that network (Parallels) are tried — only on a VM shared network, so set this for bridged networking |
| `NOTCHBAR_VM_NETS` | `192.168.64.0/24 10.211.55.0/24 10.37.129.0/24` | networks where the Mac is looked for automatically |
| `NOTCHBAR_PORT` | `47811` | |
| `NOTCHBAR_OUTPUT` | `NOTCH` | name of the invisible monitor |
| `NOTCHBAR_SCREEN` | `Virtual-1` | the built-in display's output |
| `NOTCHBAR_FOLLOW_MODE` | on under QEMU (UTM) | `1`/`0`: when UTM resizes the display to its window while running, apply and keep that size (Hyprland does not pick it up by itself) |

## Good to know

- The lock screen turns the strip black too: Omarchy draws its lock screen,
  password field included, on every output, and nothing is streamed while the
  session is locked.
- Full-screen video (or anything in real fullscreen, Super+F) turns the strip
  black, like macOS does. Maximized and tiled-fullscreen windows keep the bar.
- Omarchy's notification popups keep their usual distance below where the
  bar *would* be (a bar's height lower than panels). The notification service
  takes that distance from the bar's size and runs cloned copies sandboxed, so
  Omanotch cannot change it without editing Omarchy's own files. The popups
  never reach into the strip, though.
- Hover effects (tooltips, hover highlights) are not mirrored; clicks, right
  and middle clicks and scrolling are. Tray icons show up but can't be clicked
  in the strip.
- The bar and background clones are forks of Omarchy's plugins. After an
  Omarchy update that changes them, re-clone and run `./guest/install.sh`
  again — the patches are versioned and refuse to apply blindly.
- Hyprland warns about overlapping monitors after layout changes. The overlap
  is deliberate; `notchbar.lua` dismisses that one warning and nothing else.
- One VM at a time: while one guest is connected, another one is turned away.
- Keep UTM's library window and Parallels' Control Center out of full screen on
  the built-in display while a VM is connected: from the outside they look just
  like a full-screen VM.
- On an older Omarchy (up to about September 2026) the bar patch also makes the
  cloned bar loadable at all; newer versions don't need that.
- Cursor handling uses the window-server property `SetsCursorInBackground`,
  which is not public API (but widely used and stable for years).
- At the exact moment the pointer crosses into the strip you may catch a
  ghost of the VM cursor for a frame or two: the VM draws its own cursor and
  the display pipeline has a little latency.

## Tip: a macOS-style clock

With the bar in the menu-bar spot, a macOS-like clock at the far right feels
natural. In `~/.config/omarchy/shell.json`, move the `omarchy.clock` entry to
the end of `bar.layout.right`, give it `"format": "ddd MMM d HH:mm"`
(→ `Wed Sep 30 19:20`) and set `"centerAnchor": ""`. The shell picks the change
up by itself.

## Troubleshooting

| Symptom | Look at |
|---|---|
| Strip stays black | `~/Library/Logs/omanotch.log` ("listening on …", "guest connected"?) · in the VM: `systemctl --user status notchcast` |
| UTM: the pointer never reaches the strip | UTM's automatic input capture is on (see Requirements), or press ⌃⌥ to release the mouse |
| UTM: with capture off the VM's cursor does not move | a SPICE agent (`spice-vdagentd`) takes UTM's absolute mouse positions: it must run with a real uinput device (not `-f`) and a session agent that reports the screen size — or not at all, then QEMU's USB tablet is used |
| Bar in the VM *and* in the strip | `omarchy-shell notchbar state` → `parked` should be `true` |
| Mouse lands in the wrong place | `hyprctl monitors` → `NOTCH` must sit at the built-in display's position and width |
| Panels open on the wrong screen | `NOTCHBAR_SCREEN` must name the built-in display |

## Credits

- [Omarchy](https://omarchy.org) by DHH and contributors — the bar, the
  shell, the whole beautiful thing
- [Hyprland](https://hyprland.org) and [Quickshell](https://quickshell.org)
- Not affiliated with Omarchy, Parallels, UTM or Apple. Omarchy's bar and
  background code is not included here: it is cloned from your own Omarchy
  installation and patched at install time.

## License

MIT — see [LICENSE](LICENSE).
