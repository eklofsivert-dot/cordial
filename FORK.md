# Building this fork

This tree is a fork of [luohoa97/cordial](https://github.com/luohoa97/cordial)
(`fork` → `eklofsivert-dot/cordial`), worked on the `hyprland-pointer-lock`
branch. It is built and run on the local Arch/Hyprland machine, not in a
container.

The general build is upstream's and lives in
[`CONTRIBUTING.md`](CONTRIBUTING.md). The one thing that is easy to get wrong
here is that this client needs the **web view** features, or the link between
the browser and the client simply does not exist.

## The build

```bash
cargo build --release \
  --features cordial-shell/webview,cordial-runtime/webview
```

Both features, not one. The shell crate holds the WebKit window and
`cordial-runtime` holds the presenter that calls it; enabling only the shell's
leaves the caller compiled out, nothing references `webview::open`, the linker
collects it, and the binary carries no `libwebkitgtk` at all. That is the same
dead-code shape the justfile's `toolbox` recipe warns about.

The headers are needed to build, not just the runtime library:
`pkg-config --modversion webkitgtk-6.0` is the check that matters. On this
machine `webkitgtk-6.0` (2.52.6) is already installed, which is why the plain
host build works and no container is needed.

## Do not build through `just dev`

`just dev` runs `just build host`, which is a bare `cargo build --release` with
**no** features. It rebuilds `target/release` without WebKit and then execs
that binary, so a working web view build is silently replaced by one that opens
`openWindow` in the system browser instead. This is how the private-server Join
ended up in Firefox showing Roblox's "Download Roblox" page.

For a web view build, skip `just dev` and run the binary the build produced:

```bash
CORDIAL_DEV_CONTROL=1 \
CORDIAL_SYSTEM_PLUGIN_DIR="$PWD/plugins" \
./target/release/cordial-shell
```

`CORDIAL_AUTOSTART=1` is what `just dev --play` sets, if you want it to launch
straight in.

## Install

```bash
install -m755 target/release/cordial-shell target/release/cordial-run ~/.local/bin/
```

Stop a running client first: writing over an executing binary fails with
`Text file busy`, and a client already running keeps the old code until it is
restarted. `~/.local/bin/cordial` is a symlink to `cordial-shell` and needs no
separate install.

## Verify the build

```bash
readelf -d ~/.local/bin/cordial-run | grep -i webkit   # expect libwebkitgtk-6.0.so.4
~/.local/bin/cordial --diagnostics | head -1           # version and commit
```

In the client log, a web view build prints at startup:

```text
webview: presenter installed; an openWindow request will now be attached to the host window
```

A build without the feature prints a line beginning ``webview: built without
the `webview` feature`` and ending ``openWindow will open in your browser
instead of an in-app window``, and `readelf` finds nothing.

## Why it matters

Roblox's own web UI opens in an in-app window (`openWindow`), which is where a
game's Servers section lives. Its Join button calls
`Roblox.Hybrid.Game.launchGame`, a page-defined JS bridge that only has a
receiver inside Cordial's WebView — the shell wraps that call and turns it into
a server-specific join. In an ordinary browser there is nothing on the other
end, which is exactly why the site falls back to "Download Roblox".

## Last verified

2026-10-06, commit `f16402399` (branch `hyprland-pointer-lock`), clean tree,
host build on Arch with `webkitgtk-6.0` 2.52.6.

- `readelf -d target/release/cordial-run` → `libwebkitgtk-6.0.so.4`.
- `cordial --diagnostics` → `Cordial 0.25.0 (f16402399)`.
- Joining a private server through the game's Servers section in the in-app
  window: `hybrid launch: intercepted Roblox.Hybrid.Game.launchGame`, then
  `[deeplink] live hybrid launch: carrying placeId, joinAttemptId,
  joinAttemptOrigin, browserTrackerId, accessCode`, then `experience start`.
  The `accessCode` is what makes it the private server and not a fresh public
  one.

Not verified: whether this holds on a compositor other than Hyprland, or on a
machine building through a container.

## On a host without the headers

An immutable or sandboxed host will not have `webkitgtk6.0-devel` without
layering packages and rebooting. `just build toolbox` and `just build
distrobox` pass both features and build in a container, and the resulting
binary runs on the host because only the runtime library is needed at run time.
