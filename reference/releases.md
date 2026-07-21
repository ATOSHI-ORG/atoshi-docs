# Releases and versioning

How Atoshi tags versions, builds the `atoshid` binary, and publishes it so
node operators and the upgrade flow can fetch a verified build.

## Versioning policy

**Tags must be `v20.x.y`.** The Go module path is
`github.com/atoshi-chain/atoshi/v20`, so Go's semantic-import-versioning rules
require every release tag's major version to be **`v20`**. A tag like `v1.0.0`
is invalid for a `/v20` module — `go get` and dependency resolution ignore it.

- The next release is **`v20.3.0`** — it matches the module path *and* the
  in-code upgrade name `v20.3` (`app/upgrades/v20_3/`), so binary version, git
  tag, cosmovisor upgrade directory, and the on-chain `MsgSoftwareUpgrade`
  name all line up.
- The existing **`v1.0.0`** tag is invalid under the `/v20` module and should
  be deleted (or treated as deprecated). Do not build releases from it.

```bash
# retire the bad tag (local + remote)
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0
```

## Build from source

```bash
git clone https://github.com/atoshi-chain/atoshi
cd atoshi
git checkout v20.3.0
make build                      # → ./build/atoshid
./build/atoshid version --long  # confirm Version = 20.3.0
```

`make build` stamps the version from `git describe --tags` via ldflags
(`.../version.Version`), so building from the `v20.3.0` tag yields a binary
that reports `20.3.0`. Building off an untagged commit reports the short
commit hash instead.

## Cut a release

Tag, then publish a GitHub Release with **prebuilt binaries + checksums** so
operators don't all have to compile.

```bash
git tag -a v20.3.0 -m "v20.3.0"
git push origin v20.3.0
```

The repo already ships a `.goreleaser.yml` that builds `atoshid` for five
targets — `darwin/amd64`, `darwin/arm64`, `linux/amd64`, `linux/arm64`,
`windows/amd64` — and emits `checksums.txt`:

```bash
# requires goreleaser + a GITHUB_TOKEN with repo scope
goreleaser release --clean
```

That creates the GitHub Release for the tag and attaches the archives plus
`checksums.txt`. If you prefer to do it manually, build `linux/amd64` (the
target operators need) and upload it with a checksum:

```bash
make build
tar -czf atoshid_20.3.0_Linux_amd64.tar.gz -C build atoshid
shasum -a 256 atoshid_20.3.0_Linux_amd64.tar.gz > checksums.txt
gh release create v20.3.0 atoshid_20.3.0_Linux_amd64.tar.gz checksums.txt \
  --title "v20.3.0" --notes "…"
```

At minimum publish **`linux/amd64`** — that is what validators and full nodes
run (see [Node operation](./node-operation.md)).

## Binary URL for upgrades

A `MsgSoftwareUpgrade` proposal and cosmovisor both take an `upgrade-info`
that points at the downloadable binary for the new version. Once the release
is published, use the release asset URL:

```json
{"binaries":{
  "linux/amd64":"https://github.com/atoshi-chain/atoshi/releases/download/v20.3.0/atoshid_20.3.0_Linux_amd64.tar.gz?checksum=sha256:<sum>"
}}
```

cosmovisor downloads and swaps to this automatically at the upgrade height.
The full proposal flow is in the [Governance runbook](./governance-runbook.md#6-worked-example--software-upgrade-msgsoftwareupgrade).

## Verify a downloaded binary

```bash
# checksum must match the published checksums.txt
shasum -a 256 atoshid_20.3.0_Linux_amd64.tar.gz
tar -xzf atoshid_20.3.0_Linux_amd64.tar.gz
./atoshid version --long        # Version: 20.3.0, matching commit
```

## Release checklist

- [ ] Building from a `v20.x.y` tag (never `v1.x`)
- [ ] `atoshid version --long` reports the intended version
- [ ] Release has at least `linux/amd64` + `checksums.txt`
- [ ] For a chain upgrade: tag version, `app/upgrades/vX_Y/` handler, and the
      proposal's `--upgrade-name` all match
- [ ] `upgrade-info` binary URL points at the published asset with `?checksum=sha256:…`

## Related

- [Governance runbook](./governance-runbook.md) — software-upgrade proposals
- [Node operation](./node-operation.md) — installing the binary, running a node

---

*Last reviewed: 2026-07-22*
