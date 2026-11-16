# WittyMediaRemoteMedia

**Status: not started, on purpose.** This repository holds a decision and a design, and no program yet. Nothing
here can be installed, and no server or app offers it.

WittyMediaRemoteMedia is meant to be a small program that runs **on a NAS** (natively or in Docker) and hands a
WittyMediaServer read-only access to the media folders there: paired with the server by a code, faster and safer
than an SMB share for media, and later able to do the first look at each file on the NAS itself so a big library
scans quicker. It belongs to the [WittyMediaServer](https://github.com/wittywilla/WittyMediaServer) project
([wittymediaserver.com](https://wittymediaserver.com)).

## Why there is no program here yet

It was looked at again on 2026-11-11, with the question "could this be a small, finished module now?" The answer
is no, for reasons that are in the server, not here:

1. **A module can't add a place media lives.** Modules add *kinds* of library (`libraries` in `module.json`),
   pages, players and helpers. The module protocol has no call for "list this folder" or "give me these bytes",
   and nothing in the server would make one.
2. **The server reads media by local path, everywhere.** Each scanner walks folders with `filepath.WalkDir`;
   FFmpeg is started with a file path; direct play hands an open file to `http.ServeContent`; disc images are
   opened by path; subtitles, artwork beside the media and the metadata folder are files next to the media. A
   path that isn't on a local or mounted disk would fail in hundreds of places, not one.
3. **Modules aren't allowed on the network.** A helper only gets the `network` permission if the server's own
   list (`mayHave` in `internal/modules/service.go`) names it, and that list holds only this project's tuner, game
   and game-streaming modules, and WittyPhotos, which uses it only to hear the server on the same machine.
4. **The helper protocol is the wrong shape for media.** One call at a time, 1 MiB messages, 128 MiB blobs and a
   60-second limit suit "read this photo's EXIF", not seeking through a 40 GB film. Module API 5 lets a module
   answer HTTP routes of its own, but those answer the apps, and nothing in API 5 lets a module stand in for a
   folder the server reads, so points 1 and 2 stand.

Shipping the NAS program on its own would leave a program the server can't use. Shipping it as a read-only
WebDAV server, so the server's computer could mount it, was considered and dropped: the WebDAV clients built
into Windows, macOS and Linux are slow and fragile with large media (Windows refuses files over 50 MB until a
registry change, and seeking re-downloads), so it would be worse than the SMB or NFS share it was meant to
replace. Neither is the kind of finished feature this project ships.

## What to use today

Mount the NAS share on the server's computer (SMB or NFS), or pass it into the server's container with `-v`, and
add that path as a library. [Network shares](https://github.com/wittywilla/WittyMediaServer/blob/master/docs/LIBRARIES.md#network-shares)
in the server's docs says how, and why a `\\nas\share` path can't be chosen from another device unless allowed.
If the NAS can run Docker, running the server itself on the NAS (see the server's `DOCKER-*.md` guides for
Synology, QNAP, TrueNAS, Unraid, UGREEN and ASUSTOR) avoids the network share altogether.

## What it would take

[docs/DESIGN.md](docs/DESIGN.md) has the design: the program on the NAS, its small HTTP protocol, how it pairs
and stays read-only, and, most of the work, the list of server changes it needs first (a media file system seam
under every scanner and player, a remote source kind for library paths, pairing routes, settings rows). The
server work comes first; this program is the smaller half.

It is not a module and won't have a `module.json` or `sdui.json`: it runs on another machine and reaches the
server over the network, like the apps do. If a server-side part of it is ever packaged as a module, that part
follows the module rules of its time. Today that's module API 5, the only one the server takes: a `module.json`,
an `sdui.json` and an `integration.json` (the server's
[docs/MODULES-API5.md](https://github.com/wittywilla/WittyMediaServer/blob/master/docs/MODULES-API5.md)).

## Licence

Licensed under AGPL-3.0-or-later: see [LICENSE](LICENSE) and [NOTICE](NOTICE).

## Legal

WittyMediaRemoteMedia is an independent open-source project. It isn't associated with, sponsored by or endorsed
by any NAS maker. Any third-party names here only describe what it would work with, and belong to their owners.
It's provided "as is", with no warranty; see [LICENSE](LICENSE). Local law applies to how you use it. This isn't
legal advice.
