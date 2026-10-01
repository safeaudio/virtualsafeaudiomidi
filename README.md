[README.md](https://github.com/user-attachments/files/32927401/README.md)
# VirtualSafeAudioMIDI

Virtual audio cables and virtual MIDI cables for macOS, managed from one control panel in
your browser. Create as many as you need, change them while everything is running, and
switch the audio half and the MIDI half on or off independently.

Version 0.4.1. Read "Testing" before trusting it with a session: nothing here has run on a
real Mac yet.

## What you get

- **Audio.** Up to 16 virtual stereo audio devices, named `VirtualSafeAudio#1`,
  `VirtualSafeAudio#2` and so on. What one app plays
  into a device, another app records from it. They live in a driver (`driver/`) that macOS
  loads by itself. By default they follow the app: there while it runs, gone from macOS
  when it is closed, back when it starts. One tick in the panel makes them permanent.
- **MIDI.** Up to 16 virtual MIDI ports. Each appears in every MIDI app as an output to
  send to and as an input to receive from, under the same name; what goes in comes out.
  They live in a small helper program (`midi/`) that the panel runs while MIDI is switched
  on. Nothing to install, no password.
- **Control panel** (`gui/`). Two sections, Audio and MIDI, each with its own switch. Set
  how many devices and ports exist, rename them, save presets, watch a left/right level
  meter for every device and the MIDI traffic of every port. Every change takes effect at
  once; there is no Apply button.

You can use either half without the other.

## Start

Double-click **`START Mac.command`** in Finder. It starts the control panel and opens it in
the browser. It installs nothing. If this copy of the panel is already running it just
opens it. If a different copy or an older version is still running, it tells you which one
and offers to replace it. The version that is running is shown in the panel's header.

The first time, macOS may refuse to open a file that came from a download. In Terminal,
inside the project folder:

    xattr -d com.apple.quarantine "START Mac.command"

or use System Settings > Privacy & Security > Open Anyway. If double-clicking does nothing
at all, the file lost its executable flag on the way: `chmod +x "START Mac.command"`.

Closing the launcher's window closes the app. The virtual MIDI ports and the virtual audio
devices leave macOS and come back, unchanged, the next time you start it. To keep the audio
devices in macOS all the time, tick *Keep devices when the app is closed* in the panel.

Both halves need the Xcode command line tools once, to compile: `xcode-select --install`.
macOS 11 or later.

## Audio

**Installing the driver.** In the panel's Audio section press *Install audio driver* and
enter your Mac password in the dialog macOS shows. Or from Terminal: `./install.sh`. Either
way all sound on the Mac drops for a second or two while Core Audio restarts. That is the
only time Core Audio is restarted. `./uninstall.sh` removes the driver.

**How many devices.** Set *Number of stereo devices* at the top of the Audio section, or use
*Add a device* and each row's *Remove device*. macOS follows at once: raise the number to 4
and `VirtualSafeAudio#1` to `#4` are in the sound menu; lower it to 0 and none is. Nothing
is restarted, and the other devices keep playing while one is added or removed.

**Names and numbers.** A new device takes the lowest free number, so after removing `#2`
the next one added is `#2` again, and apps that were using `#2` recognise it. You can type
your own name into a device's field (press Return). The device keeps its identity, but it
reconnects for a moment: macOS reads a device's name when the device appears, so the app
takes the renamed device out and puts it straight back. Other devices are not disturbed.

**Stereo.** Every device has two inputs and two outputs. The driver underneath could carry
more channels per device, but the app does not offer it.

**When the app closes.** With *Keep devices when the app is closed* unticked (the default),
the audio devices exist only while the app runs. Closing the launcher window, pressing
control-C, or even force-quitting the app removes them from macOS within a few seconds; a
small guard process that outlives the panel sees to that. The layout is saved and returns
at the next start under the same identities, so a DAW finds its devices again. The one
case it cannot cover is the Mac losing power: the devices are then still listed after the
restart until the app has been started and closed once. With Keep ticked, the devices stay
in macOS whether or not the app runs, including after a reboot.

**The Audio switch.** Off means the driver publishes no devices; the panel remembers the
layout and gives it back when you switch on again. The driver stays installed.

Sample rate is not set in the panel. Like any audio device it follows the app using it, or
Audio MIDI Setup (44.1 to 192 kHz), and the driver remembers it.

## MIDI

Switch MIDI on and set *Number of ports* (or use *Add a port*). Ports are named
`VirtualSafeMIDI#1`, `#2`, ... and can be renamed, like the audio devices. The first
switch-on compiles the helper, which takes a few seconds.

- A port is a loopback: send to it in one app, receive from it in another. Notes, clock,
  system exclusive, everything passes through unchanged and is not re-timed.
- Adding, removing or renaming a port does not disturb traffic on the other ports.
- Each port keeps the same identity from run to run, so a DAW that was pointed at a port
  finds it again after the panel has been restarted.
- The panel shows a lamp and a messages-per-second figure for each port.
- As with any loopback port: an app that listens to a port and echoes what it hears back to
  the same port creates a feedback loop. Switch off MIDI thru for that port in that app.

**The MIDI switch.** Off stops the helper and the ports disappear; the list of ports is kept
for next time.

## Presets and settings

A preset stores the audio devices and the MIDI ports together. Loading one makes it real at
once, for whichever halves are switched on.

Files, in `~/Library/Application Support/VirtualSafeAudioMIDI/`:

- `presets.json`: your presets.
- `settings.json`: which halves are switched on, the Keep choice, the MIDI port list, and
  the audio layout to bring back when the app starts or audio is switched on again.

## Panel options

    python3 gui/virtualsafeaudiomidi_gui.py --open          # what the launcher runs
    python3 gui/virtualsafeaudiomidi_gui.py --mock --open   # preview: nothing real is touched

`--port N` changes the port (default 8767). `--host 0.0.0.0` makes the panel reachable from
other machines on your network; there is no password, so only do that on a network you
trust.

## How the pieces talk

    browser  <-- HTTP + server-sent events -->  panel (Python)
                                                  |-- Core Audio properties -->  audio driver (inside coreaudiod)
                                                  '-- text over stdin/stdout ->  MIDI helper (child process)

**Audio driver** (`driver/virtualsafeaudio.c`). Two custom properties on its plug-in object:
`'vscf'` (read/write) is the device list, one line per device,
`uid <TAB> channels <TAB> sampleRate <TAB> name`; `'vsmt'` (read only) is the current levels,
`uid <TAB> peak0,rms0,peak1,rms1,...`. Levels are measured inside the driver, so the panel
never opens a device for recording: no microphone permission, no orange recording dot.

**MIDI helper** (`midi/vsamidi.c`). The panel writes the wanted ports, one `uid <TAB> name`
per line, then a line with a single dot. The helper answers `OK` with the ports that exist
and reports traffic as `A <TAB> uid=count ...`. MIDI itself never passes through the panel
or through Python: the helper forwards it on CoreMIDI's own thread. If the panel goes away,
however abruptly, the helper notices and exits.

Both source files open with a plain-language explanation of how they work.

## Testing

The test plan is `VIRTUALSAFEAUDIOMIDI-STP-v0.4.md`: 234 tests in 8 chapters plus a results
chapter. It is generated from `stp/results.json` by `stp/stp.py`, which also runs the tests.

    python3 stp/stp.py run                     # run every automated test this machine can run
    python3 stp/stp.py pending                 # manual tests that still have no result
    python3 stp/stp.py set MID-029 pass        # record a manual result
    python3 stp/stp.py set AUD-012 fail "what happened"
    python3 stp/stp.py run --after-uninstall   # on the Mac, right after ./uninstall.sh

Three kinds of test:

- **Automated (120).** The driver source and the MIDI helper source compiled against
  stand-ins for Apple's headers and driven the way macOS and other apps would drive them
  (also under AddressSanitizer and UBSan); the panel server against a mock driver and the
  real helper linked to a pretend CoreMIDI; the page in a headless browser (needs
  Playwright); the launcher. All pass. These were run on Linux, where the project was
  written.
- **Mac runner (34).** The same `run` command on the Mac. It builds everything and checks
  the bundle; with the driver installed it creates, renames, resizes and removes its own
  test audio devices and checks that macOS lists them, plays speech into one and reads the
  meters; for MIDI it opens test ports and sends 100 notes through one from the outside,
  the way another app would. It restores your layout when it finishes. Not yet run.
- **Manual (80).** Things only a person with real apps can judge: Audio MIDI Setup, DAWs,
  null tests, sample rates, MIDI clock, SysEx, soak, sleep/wake, browsers. Not yet run.

First contact with a real Mac (1 October 2026): the launcher and the panel run, the driver
compiles, signs, installs and is loaded by macOS, its device appears in the sound menu, and
the panel reaches it. The panel's install button failed with the
project in Downloads (fixed in 0.2.1, needs retest; 0.2.2 makes sure the fixed version is the one that runs). No audio has
yet been passed through a device, and MIDI has not been tried.

## Troubleshooting

**Audio: no "VirtualSafeAudio" device after install.** Confirm the bundle is in place and
signed:

    ls /Library/Audio/Plug-Ins/HAL/
    codesign -dv /Library/Audio/Plug-Ins/HAL/VirtualSafeAudio.driver

then look at what Core Audio said while loading it:

    log show --last 3m --predicate 'process CONTAINS "coreaudio" OR process CONTAINS "Core-Audio-Driver"' | grep -i -B2 -A2 virtualsafeaudio

If the log points at the signature, re-sign with a real identity (a free Apple Development
certificate is enough for your own machine) and install again:

    codesign --force --sign "Apple Development: you@example.com (TEAMID)" build/VirtualSafeAudio.driver
    ./install.sh --no-build

**Audio: the device is there, the panel says it cannot reach the driver.** The device
itself works; the panel cannot find the plug-in object. Run
`python3 gui/virtualsafeaudiomidi_gui.py` in Terminal and look at what it prints.

**MIDI will not switch on.** The panel shows the reason next to the MIDI switch. Most
likely the helper did not compile; run `./build.sh midi` in Terminal to see the compiler's
own words.

**Removed every audio device by accident.** Add one in the panel. (Reinstalling does not
reset the list; Core Audio keeps the driver's settings.)

## Not in 0.2

- MIDI ports that stay when the app is closed (audio devices can, with Keep). That needs
  the helper to run as a login item in the background.
- MIDI 2.0. The helper uses CoreMIDI's MIDI 1.0 packet calls, which carry everything MIDI
  1.0 gear sends.
- Volume and mute controls on the audio devices.
- Audio devices with more than two channels. The driver can do it; the app makes stereo only.
- An installer package, Developer ID signing and notarization, which are only needed to
  give the app to other people.
