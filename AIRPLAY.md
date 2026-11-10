# AirPlay sessions as shairport-sync reports them

What shairport-sync 5.6.2 (with NQPTP 1.2.9) reports over D-Bus while an iOS
device plays to it over AirPlay 2, and what that means for audiocontrol. Observed
on a test device in November 2026, with the Music and Podcasts apps, by
polling the D-Bus properties and the receiver's TCP connections once a second.
Treat the timings as observations, not guarantees: AirPlay 2 remote control is
new upstream (5.4-dev onwards) and upstream calls it "evolving".

## Two interfaces

| Bus name | Used for |
|---|---|
| `org.mpris.MediaPlayer2.ShairportSync` | MPRIS: what audiocontrol reads and controls |
| `org.gnome.ShairportSync` | native interface: raw AirPlay 2 data, useful for diagnosis |

Both are on the system bus. On the native interface, `NowPlayingInformation`
and `CommandInformation` (interface `org.gnome.ShairportSync.Client`) are the
sender's raw now-playing data and its list of accepted remote commands.

## MPRIS `PlaybackStatus` follows the receiver, not the sender

`PlaybackStatus` reflects shairport-sync's own player thread. It reads
`Stopped` whenever that thread is not running, which happens both when the
session has ended and some time into a pause:

| What happens on the iPhone | `PlaybackStatus` | `Active` (native) | AirPlay connection (port 7000) |
|---|---|---|---|
| Receiver selected, nothing played yet | `Stopped`, empty title | false | open |
| Play | `Playing` | true | open |
| Pause | `Paused` at once | false after about 10 s | open |
| … still paused | `Stopped` 10 to 30 s into the pause | false | **open** |
| Output switched away (playing or paused) | `Stopped` | false about 10 s later | **closed at once** |

Consequences:

- `Stopped` does not mean the session ended. A paused sender turns into
  `Stopped` with the connection still open, and pressing play resumes it.
- After the session ends, `Metadata` keeps the last song. A reader that wants
  "nothing playing" must clear it itself.
- Pausing through MPRIS behaves the same way as pausing on the iPhone.
- Nothing on D-Bus tells "paused for a while" from "disconnected": of the 63
  properties on both interfaces, only `NowPlayingInformation` and `Position`
  differed between the two states.
  - `RemoteControl.Client` keeps the last sender's address after it has
    disconnected, so it is no session signal.
  - `RemoteControl.Available` is false and
    `AdvancedRemoteControl.PlaybackStatus` is `Not Available` throughout:
    these belong to the AirPlay 1 (DACP) remote control.
- The open TCP connection on the AirPlay port is the only reliable signal
  found: it stays open through any pause and closes as soon as the sender
  switches its output away.

audiocontrol works around this with `session_ports` in
`/etc/audiocontrol/players.d/shairport.json` (default `[7000, 5000]`, AirPlay 2
and AirPlay 1): while shairport-sync reports `Stopped` but a connection is open
on one of these ports, the song stays and the player shows as paused. `[]`
turns this off. If shairport-sync is configured to listen on other ports,
change the list to match.

## Remote control from the receiver

shairport-sync forwards MPRIS `Play`, `Pause`, `PlayPause`, `Stop`, `Next` and
`Previous` to the AirPlay 2 sender as MediaRemote commands 0 to 5, and
`Shuffle`/`LoopStatus` as commands 26 and 25. What the sender then does is up
to iOS and the app:

- **Next with nothing queued pauses.** When the sender's queue holds only the
  current song (`TotalQueueCount` 1 in `NowPlayingInformation`), Next stops
  playback instead of skipping. Playing a song on its own from an album gives
  a queue of one; playing the album or a playlist queues the rest.
- **Previous restarts the song** when it has been playing for more than a few
  seconds; a second Previous right after goes back a track. This is the same
  as the Previous button on the iPhone.
- **Some apps ignore skipping.** In the Podcasts app, Next and Previous had no
  effect.
- `CanGoNext`, `CanGoPrevious` and `CanPlay` read true even when the command
  will do nothing; `CanPlay` is true with no sender connected at all.
  `CommandInformation` shows which commands the sender enables, but command 4
  (Next) is reported as enabled even with a queue of one.

## Cover art

shairport-sync writes artwork to `/tmp/shairport-sync/.cache/coverart` and
reports it in `mpris:artUrl` as a `file://` URL; with no artwork it reports
`file://(null)`. The directory must be writable by the user shairport-sync
runs as. A directory left behind by another user (older audiocontrol versions
created it themselves in UDP mode) silently stops artwork: shairport-sync then
reports `file://(null)` for every song. Removing the directory fixes it.

## Reading the state by hand

```sh
busctl --system get-property org.mpris.MediaPlayer2.ShairportSync \
    /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player PlaybackStatus
busctl --system get-property org.gnome.ShairportSync \
    /org/gnome/ShairportSync org.gnome.ShairportSync.Client NowPlayingInformation
ss -tn '( sport = :7000 )'
```
