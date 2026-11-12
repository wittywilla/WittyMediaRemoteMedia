# WittyMediaRemoteMedia: design

Written 2026-11-11, against WittyMediaServer 0.8.0-alpha. Nothing here is built. It is the plan a later session
starts from, and the list of what the server needs first. File names below are the server's.

## What it is for

People keep media on a NAS and run the server on another computer. Today the server reaches it through an SMB or
NFS share, which brings three troubles the server's own docs already warn about:

- On Windows, reaching a share makes the server sign in there with its own account, so a share can't be chosen
  from another device unless `allowNetworkPaths` is set (`errOtherMachine` in `internal/server/admin_paths.go`).
- A share gives the server more than it needs: write access, every folder the account can see.
- The first scan of a big library on a slow share is slow, because every file is opened and probed over the
  network.

WittyMediaRemoteMedia answers those with a small program on the NAS that offers chosen folders, read-only, to one
or more paired servers.

## What it isn't

- **Not a module.** It runs on another machine and the server reaches it over the network, as the apps do. It is
  released like an app (a program per platform and a Docker image), not through the module store, and has no
  `module.json` or `sdui.json`.
- **Not a file sharing tool.** It answers paired servers only, never a browser, and never writes.
- **Not a way round the server's rules.** Encrypted discs stay refused by the server's own code; account limits,
  parental controls and library access are all decided by the server as they are for local files.

## Why it can't be a module (checked 2026-11-11)

| Fact in the server | Where | What it means |
|---|---|---|
| Modules add kinds of library, pages and helpers, never a place files live | `docs/MODULES.md`, `wittymodule` manifest | There's no call a module could answer to stand in for a folder |
| Scanners walk folders by path | `filepath.WalkDir` in `internal/library/scan.go`, `audio.go`, `books.go`, `cd.go`, `creators.go`, `games.go`, `photos.go` | Every scanner would need another way to walk |
| FFmpeg is given a file path | about 27 `"-i"` sites: `internal/ffmpeg`, `internal/hls`, `internal/libtranscode`, `internal/bdsession`, `internal/library/audio.go`, `internal/server/api_cd.go` | Playback and probing need a URL or a pipe instead |
| Direct play serves an open file | `http.ServeContent` / `ServeFile` in `api_play.go`, `api_music.go`, `api_music_home.go`, `api_photos.go` and others | Needs a ranged reader instead of `*os.File` |
| Disc images are opened by path | `isofs.Open(path)` in WittyDisc; WittyDVD, WittyBluray and WittyCD helpers read by path | Needs a reader-at, and helpers can't reach the network |
| About 470 `os.Open`, `os.Stat`, `os.ReadDir`, `filepath.WalkDir` calls outside tests | `internal/` | Sidecars, artwork beside media, subtitles, the metadata folder, game archives |
| Only listed modules get `network` | `mayHave` in `internal/modules/service.go` | A new or forked module can't open a socket |
| Helper calls are one at a time, 1 MiB messages, 128 MiB blobs, 60 s | `docs/MODULES.md`, the helper protocol | Wrong shape for streaming and seeking large files |

So the work is mostly in the server, and has to come first.

## The program on the NAS

A single Go program with no cgo and no dependencies outside the standard library, built for linux/amd64,
linux/arm64, linux/arm (v7), freebsd/amd64 (TrueNAS CORE), and the usual desktop systems for testing, plus a
small Docker image (`FROM scratch`, the binary and CA certificates).

```
wittyremotemedia serve -config /config/remotemedia.json
wittyremotemedia pair            # prints a 6-digit code, valid 10 minutes
wittyremotemedia servers         # lists paired servers; `unpair <id>` removes one
```

**Configuration.** A JSON file names the folders offered, each with a short name (`films` → `/volume1/Films`),
the address to listen on (default port 9790), and where its key and paired servers are kept. Nothing is offered
until a folder is named.

**Read-only, and boxed in.** Each folder is opened with `os.OpenRoot` (Go 1.24 and later), so `..`, absolute
paths and symlinks that leave the folder are refused by the kernel-backed root, not by string checks. The program
opens files only for reading and has no call that writes, renames or deletes. In Docker the folders are mounted
`:ro` as well.

**Identity and pairing,** reusing what the server already does for its own identity (`internal/identity`,
"WittyMediaServer identity v1" in `docs/CLIENT-API.md`):

1. The program makes an Ed25519 key on first start and a self-signed TLS certificate from it.
2. `wittyremotemedia pair` shows a 6-digit code and the key's fingerprint.
3. An admin types the address and code in the server's settings. The server calls `POST /v1/pair {code,
   serverId, serverName}` over TLS, without trusting the certificate yet; the program answers with a random
   256-bit token for that server and its public key, signed together with a nonce the server sent.
4. The server pins that public key, as apps pin a server's. Every later connection checks the certificate's key
   against the pin before the token is sent. A mismatch stops and says a different machine is answering.
5. Five wrong codes in a row end the pairing window. Codes are single use.

**Protocol** (HTTPS, JSON, `Authorization: Bearer <token>`, everything under `/v1/`):

| Call | Answer |
|---|---|
| `GET /v1/hello` | Program version, protocol version, the folders offered (name, total size if cheap) |
| `GET /v1/list?folder=films&path=Some%20Film&cursor=` | One directory: name, kind, size, mtime per entry; pages of 1,000 |
| `GET /v1/stat?folder=films&path=...` | One entry, or 404 |
| `GET /v1/file?folder=films&path=...` | The bytes, with `Range`, `ETag` (size + mtime) and `Last-Modified`; served by `http.ServeContent` from the root |
| `GET /v1/changes?since=<cursor>` | What changed since a cursor, from a periodic walk the program does at low priority; long-polls up to 60 s |

Later, as the roadmap says: `GET /v1/probe` (FFmpeg's view of a file, done on the NAS when it has FFmpeg) and
`GET /v1/thumb`, so the first scan doesn't pull every file over the network.

**Limits for small NAS boxes.** At most 16 open files and 8 concurrent requests per paired server, a 64 KiB
copy buffer, walks paced with sleeps between directories, no index kept in memory beyond one directory page.
Memory stays under about 30 MB.

**Tests the program would ship with.** Path escapes (`..`, absolute, symlink out, Windows drive and UNC forms)
refused; ranged reads match the file; ETag changes when a file changes; wrong, missing and old tokens refused;
pairing codes single use and limited to five tries; the change feed sees a new, changed and removed file; a
paired server's key mismatch is detected (server side). All with generated files in a temporary folder.

## What the server needs first

This is the larger half, and none of it exists. In order:

1. **A media file system seam.** A small interface in a new package (for example `internal/mediafs`):
   `Open` (returning an `io.ReadSeekCloser` with `io.ReaderAt`), `Stat`, `ReadDir`, `WalkDir`, plus `URL` for
   FFmpeg. A local implementation that wraps `os` exactly as today, and a remote one over the protocol above. Every
   scanner, the direct-play handlers, artwork and subtitle readers move onto it. This is the big, careful change:
   about 470 call sites, most of them mechanical, with the local path behaviour unchanged and covered by the
   existing tests.
2. **A source kind for library paths.** A new migration adding `source` to `library_paths` (`local` by default;
   `remote:<agentId>` with a folder name and path otherwise). Item paths keep their meaning inside a source.
3. **FFmpeg input over HTTP.** For a remote file the server gives FFmpeg a loopback URL of its own
   (`http://127.0.0.1:<port>/internal/mediafs/<short-lived signed ticket>`) that it serves from the remote reader,
   so the NAS token never appears on a command line or in a log. Probing, HLS, transcoding and library
   transcoding all take that URL. (Library transcoding writes its output beside the media, so it must be refused,
   or write to the server's own storage, for remote sources.)
4. **Disc images.** `isofs` needs an `OpenReaderAt`; the DVD, Blu-ray and CD helpers read by path and have no
   network, so for a first version disc images and disc folders on a remote source are refused, with a message
   that says why, until the helpers can be handed a reader through the server.
5. **Nothing written beside remote media.** "Save information beside the media" (`saveMetadata`), sidecar
   artwork, canvases, stems and anything else that writes next to a file is forced off for remote sources and
   kept in the server's data folder instead.
6. **Offline handling.** The scanner already counts unreachable roots as `offline` and keeps their items
   (`internal/library/scan.go`, `markMissing`). A remote source that doesn't answer uses that same path; playback
   says the NAS isn't answering instead of "file not found".
7. **Change detection.** Scans of a remote source read `/v1/changes` from the last cursor instead of walking the
   whole tree, falling back to a full walk if the cursor is too old.
8. **Pairing and management routes** (admin only): `GET/POST /api/v1/remote-media` (list, pair with address and
   code), `DELETE /api/v1/remote-media/{id}`, `GET /api/v1/remote-media/{id}/folders` (for the folder picker).
   Each needs its rule in `route_rules_test.go`, a parental classification (admin only, not reachable by limited
   profiles), `docs/openapi.json`, `docs/SECURITY-ROUTES.md`, and a feature name in `serverFeatures`.
9. **Settings, hand-built.** A **Remote media** row under the existing libraries settings in the web
   (`web/src/pages/settings/`) and the apps' settings screens, in the existing style: paired NAS boxes, pair one,
   remove one; and in the add-folder picker, a choice between this computer and a paired NAS. SDUI does not
   replace this.
10. **WittyDiagnoser.** A check for each paired NAS: answering, key matches the pin, round-trip time, read speed.
11. **Docs.** `docs/LIBRARIES.md` (a "Remote media" section beside "Network shares"), `docs/CONFIG.md`,
    `docs/TROUBLESHOOTING.md`, the CHANGELOG, and `docs/REPOSITORIES.md` / `docs/ROADMAP.md` updated.

## Order of work and size

1. Server seam (step 1) with the local implementation only, no behaviour change: the riskiest part, worth doing on
   its own and merging once the existing tests pass. Several days of careful work.
2. This program, with its tests and Docker image: two or three days.
3. Server steps 2–3 and 5–8, with tests against a fake remote served by `httptest`: a few days.
4. Settings rows (step 9) in the web, then the apps, and the diagnoser check.
5. A real run against a Linux NAS or a Linux VM with a large generated library, timing a first scan against SMB.

Probe and thumbnails on the NAS (the "Later" in the roadmap) come after all of that works.

## Decisions for the owner

- **Port.** 9790 is proposed. Not 9798 and up: the server uses those for a second copy on one machine
  (`FirstOtherPort` in `internal/instance/instance.go`). It must not clash with anything the NAS systems use by
  default.
- **Disc images on a NAS.** Refused in the first version (step 4), or worth the extra helper work up front?
- **Where it's released.** As an app (GitHub releases plus a Docker image), listed on the website's projects page
  as a program rather than a module.
