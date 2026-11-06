# Xiaomi `miss` cold-start fixes (go2rtc fork)

This fork is [AlexxIT/go2rtc](https://github.com/AlexxIT/go2rtc) `master` (`990f4de3cc`,
v1.9.15) plus a small, self-contained patch to `pkg/xiaomi/miss/**`. It improves
cold-start reliability for battery-powered Xiaomi `miss` camera doorbells that use
the `cs2` transport and emit HEVC. No changes outside `pkg/xiaomi/miss/**`.

## Why

On a sleeping doorbell the LAN discovery response arrives late and codec parameter sets
are split across access units, sometimes prefixed by AUD/SEI. The upstream probe waits
for a single access unit containing VPS and then for an audio packet that a muted door
card never sends; cold starts time out or open with a black picture. A ticker-based
command ACK can also be kept alive by a late ACK from a previous command.

## What changed

- `cs2/conn.go`: command ACK lifecycle via a buffered channel guarded by a mutex
  (replaces the ticker + atomic that a late ACK could restart); an auth response counts
  as delivery proof only while an auth request is pending; discovery and readiness
  negotiation get separate bounded deadlines.
- `producer.go`: collect HEVC VPS+SPS(+PPS) / H.264 SPS+PPS across access units,
  tolerate AUD/SEI prefixes, release as soon as parameter sets are complete without
  waiting for audio, and replay buffered initial media (4 MiB cap) so the first
  keyframe is not lost during probing.
- Three regression tests: cold discovery recovery, ACK lifecycle (late/unrelated ACK),
  parameter-set splitting without audio.

The exact diff is also provided as [`go2rtc-cold-start.patch`](go2rtc-cold-start.patch)
at the repository root (standard `git apply` format).

## Build / verify

```sh
git clone https://github.com/Hylouis233/go2rtc
cd go2rtc
CGO_ENABLED=0 go build ./...
CGO_ENABLED=0 go test -count=1 ./pkg/xiaomi/miss/...
```

If you prefer to patch your own checkout instead of using this fork:

```sh
git apply go2rtc-cold-start.patch   # against go2rtc master 990f4de3cc
CGO_ENABLED=0 go test -count=1 ./pkg/xiaomi/miss/...
```

## Scope / limits

- Cloud authorization and wake are still required each session (per the upstream
  Xiaomi source README); only the LAN media path after wake is changed here.
- Media stays LAN-only. This patch does not add polling, keep-alives, or scheduled wakes.
- Verified against upstream `master` as of 2026-11-05. The tests are unit-level
  regression tests; a single real-device cold-start sample was measured during
  development, not a long-play stability guarantee.

## License / attribution

go2rtc is MIT licensed by AlexxIT; this patch is released under the same MIT terms.
The patch was prepared for general use with no device identifiers, account data,
private network addresses, or environment-specific paths included.
