# kari-ios

Stream anime on your **iPhone / iPad** from the terminal and watch it in **VLC**, no Mac and no downloading needed.

It's a POSIX-shell port of the anime part of [kari](https://github.com/dhairya3391/kari), made to run in [iSH](https://ish.app) the same way [ani-cli](https://github.com/pystardust/ani-cli) used to.

```
$ kari frieren
Select anime: Frieren: Beyond Journey's End  (TV, 2023) [28 eps]
Episode (1-28): Episode 1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~  Tap here to open in VLC  ~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

## What you need

- **iSH Shell** (free, App Store)
- **VLC for Mobile** (free, App Store)

## Install

Open iSH and paste:

```sh
apk add curl
curl -sL https://raw.githubusercontent.com/AeroDyn4m1c/kari-ios/main/install.sh | sh
```

This installs the dependencies (`curl jq openssl fzf`) and puts `kari` in `/usr/local/bin`.

<details>
<summary>Manual install</summary>

```sh
apk add curl jq openssl fzf
curl -sL https://raw.githubusercontent.com/AeroDyn4m1c/kari-ios/main/kari -o /usr/local/bin/kari
chmod +x /usr/local/bin/kari
```
</details>

## Use

```sh
kari                 # start menu (see below)
kari frieren         # search directly
kari -d one piece    # English dub
kari -e 5 frieren    # start at episode 5
kari -c              # continue watching (history)
kari -w dandadan     # add to the watchlist (kari -w alone opens it)
kari -f frieren      # follow an anime for news (kari -f alone opens the list)
kari -n              # news about the anime you follow
kari -D frieren 1-12 # download episodes 1-12 (see "Download for offline")
kari -u              # update kari
kari -U              # install the test version (v2 branch); kari -u goes back
kari -h              # all options
```

1. Pick the anime and the episode. Type to filter, then tap the item or press Enter.
2. Wait for **"Ready: first segment ..."**. kari checks that the video really comes through before it shows the link.
3. **Tap the green "Tap here to open in VLC" box.** VLC opens and starts playing, with English subtitles in sub mode.
4. When you're done, switch back to iSH and press **Enter**, then choose **next**, **replay**, **previous**, **select**, **switch to dub/sub**, **quality**, **search**, **start menu** or **quit**.

### First run: allow location access

The first time you play something, iOS asks whether **iSH** may use your location. Choose **Allow While Using App** (or **Always**).

kari doesn't use your location. Reading it is the standard trick that keeps iSH running in the background, and iSH needs to keep running while you watch (see below). If you say no, playback stops after a few seconds once VLC is in front.

## Download for offline

```sh
kari -D frieren 5        # one episode
kari -D frieren 1-12     # a range
kari -D frieren 13-      # episode 13 to the last one
kari -D -d frieren all   # every episode, dubbed
kari -D frieren          # pick the season, then type which episodes
kari -D                  # asks for the anime, season and episodes
kari -c -D 6-10          # more of an anime from your history
kari -D -o ~/videos frieren 1-3   # save somewhere else
```

You always pick the season (AniList lists each season separately) before kari downloads. The episodes go after the title; if a title itself ends in a number, put it in quotes: `kari -D "mob psycho 100" 3`.

Each episode is saved as `<title> - Episode 05 (sub).mkv`, with the subtitles inside the file. A 24-minute episode is about 180-250 MB.

**One-time setup: connect VLC's folder to iSH.** Downloads go to `/mnt/vlc`, which kari expects to be VLC's own folder, so they show up straight in VLC's library:

```sh
mkdir -p /mnt/vlc && mount -t ios . /mnt/vlc
```

In the window that opens, pick **On My iPhone** (or **On My iPad**) › **VLC**. iSH may forget this after it restarts; kari tells you when to run the `mount` line again.

**For .mkv files, install ffmpeg once:** `apk add ffmpeg` (about 100 MB). Without it, kari saves a `.ts` video with a separate `.vtt` subtitle file next to it.

Downloading needs iSH to keep running, like playing does: keep it open, or allow location access (see above) and switch to other apps. If a download is interrupted, run the same command again: it continues where it stopped, and skips episodes that are already saved.

## Start menu, watchlist and news

`kari` on its own opens the start menu:

```
kari 2.0.0
  1 JUJUTSU KAISEN - next: ep 7/24 (sub)
  2 ONE PIECE - next: ep 1101/1179 (sub)
  w Watchlist (3)
  f Following (4)
  n News (2 new!)
  h All history
Number, letter, or an anime name to search (Enter quits):
```

- **A number** plays the next episode of that anime. The list shows the 5 most recent ones you haven't finished.
- **Type a name** and press Enter to search, the same as `kari <name>`.
- **w, the watchlist:** anime you want to watch. Type a name to add it, `d2` removes entry 2, and a number searches that entry as if you ran `kari <name>`, so you pick the season there. An entry leaves the watchlist once you start watching it.
- **f, following:** when you start the last episode out, kari asks whether to follow that anime. In this list, type a name to search and follow any anime (also ones you watched outside kari), `a` adds one from your history, and `d2` stops following entry 2.
- **n, news:** checks everything you follow in one request and lists only what's new: new episodes, new seasons, and movies, specials and OVAs. Unwatched seasons stay listed until you watch them or dismiss them. Movies and specials that already existed when you followed the anime don't count as news. If nothing changed, it says "Nothing new." kari also checks once a day by itself, the first time you open the start menu.
- **h:** your whole history, the same as `kari -c`.

## How it works

kari follows the same anime pipeline as the original kari's AniCine provider:

1. Search **AniList** for the title, which gives the episode count and MAL id.
2. Open the **Megaplay** embed for `MAL id / episode / sub|dub`.
3. Ask Megaplay for the sources and decrypt the AES token with `openssl`, which yields the HLS playlist.
4. Sign the CDN URL (HMAC-SHA256), as kari does.

The stream's CDN refuses any request without a `Referer` header, and VLC for iOS can't send one. On iSH, kari therefore downloads the video segments itself (with the header), a little ahead of where VLC is, and serves them on `127.0.0.1:8642` with a tiny server (busybox `nc` running a shell script per request). VLC plays `http://127.0.0.1:8642/...`. When VLC asks for a part that isn't downloaded yet, for example after skipping ahead, the server waits for it instead of answering "not found", because VLC would treat that as the end of the video.

A background job downloads the next ~80 seconds of video ahead of VLC, with the right headers and one connection per batch, because every new connection is slow inside iSH. Segments VLC has already played get deleted, so storage use stays small. All of this is why iSH has to stay alive in the background while VLC plays.

On a normal Linux/macOS machine kari skips the proxy and starts `mpv` or `vlc` directly with the headers.

## Settings

Set these as environment variables, e.g. in `~/.profile`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `KARI_MODE` | `sub` | `sub` or `dub` |
| `KARI_QUALITY` | `best` | `best`, `worst`, `1080`, `720`, `480`, `360` |
| `KARI_SUB_LANG` | `English` | subtitle language (`Spanish`, `French`, ...) or `none` |
| `KARI_PORT` | `8642` | local proxy port |
| `KARI_PLAYER` | auto | `iSH`, `mpv`, `vlc`, or `link` (just print the stream URL) |
| `KARI_NO_FZF` | | set to use a numbered menu instead of fzf |
| `KARI_NO_KEEPALIVE` | | set to skip the location background trick |
| `KARI_AHEAD` | `20` | how many segments (a few seconds each) to buffer ahead of VLC |
| `KARI_DOWNLOAD_DIR` | `/mnt/vlc` on iSH | where `-D` saves episodes |
| `KARI_DL_PARALLEL` | `4` | pieces downloaded at once by `-D` (`1` = one at a time) |
| `KARI_DL_CHACHA` | `1` | `0` turns off the faster ChaCha20 encryption for `-D` |
| `KARI_DEBUG` | | set to show how long each step takes |

History is stored in `~/.local/state/kari/history`.

## Troubleshooting

- **VLC opens but nothing plays / stops after a few seconds.** iSH was suspended. Allow location access for iSH (Settings → iSH → Location → **Always**), and don't force-close iSH while watching. To see what the proxy did, run `cat ~/.cache/kari/proxy.log` (every video segment kari downloaded) and `cat ~/.cache/kari/requests.log` (every request VLC made).
- **"Can't connect to the video server"**: the video CDN did not answer within 20 seconds. Your Wi-Fi or internet provider may be blocking it; try another network.
- **Skipping ahead pauses for a few seconds.** kari only downloads a little ahead of VLC, so after a skip it first downloads the new part; VLC waits and then continues.
- **"Warning: test download through the proxy failed"**: kari still shows the VLC link, so try it. If VLC does not play, report the warning line together with `cat ~/.cache/kari/proxy.log`.
- **Slow to start.** Run `KARI_DEBUG=1 kari frieren` to see which step takes long. Lower quality (`kari -q 720 ...`) also buffers faster.
- **"Episode N (dub) is not available"**: that episode has no dub yet. Switch to sub.
- **"Could not download the first video segments"**: the video server didn't deliver the start of the episode within 2 minutes. `proxy.log` shows the curl error for each segment.
- **"Local proxy failed to start"**: another program is using the port. Run `KARI_PORT=8700 kari`. `kari --test-proxy` checks what the local server can serve on your device.
- **Nothing tappable appears**: update iSH. Older versions don't support terminal links.
- **Search or playback suddenly stops working**: the upstream site probably changed. Run `kari -u` to get the latest fixes.

## Credits

- [kari](https://github.com/dhairya3391/kari) by Dhairya (MIT), for the provider logic and Megaplay keys this is ported from.
- [ani-cli](https://github.com/pystardust/ani-cli), for the iSH and VLC approach.

This tool doesn't host any content; it only links to streams that are already public.

## License

MIT. See [LICENSE](LICENSE).
