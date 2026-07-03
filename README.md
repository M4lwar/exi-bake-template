# exi-bake-template

A template repo that bakes a schema variant of
[`libexificient`](https://github.com/M4lwar/exificient-native-image) —
GraalVM Native Image build of the EXIficient EXI codec — **without forking
the library**. It works by calling the library's reusable bake workflow and
handing it your `.xsd`; the library repo itself never changes.

## What this is

`libexificient` ships schema-neutral: every `exi_create(schemaPath)` call
parses and compiles the schema's grammars at runtime. For a small test
schema that's cheap; for a large real-world schema family it isn't — the
UCI 2.5.0 worked example below (schema is ~8.6 MB across two `.xsd` files)
takes on the order of **14 seconds** to cold-load via `exi_create`.

Baking compiles the schema's grammars into the native image at build time.
The exact same message content then decodes/encodes against a context
created with `exi_create(NULL)`, and creating that context costs **~4
milliseconds** instead of ~14 seconds — the schema-informed grammar tables
are already resident in the binary's image heap. This repo is a worked,
runnable example of doing that bake for your own schema, driven entirely by
CI: you never build GraalVM Native Image locally.

The whole "config" is two values (the schema path and an id for the
variant); everything else is the library's reusable bake pipeline.

## Use this template (GitHub)

1. Click **Use this template** (or fork) to create your own copy of this
   repo.
2. Drop your schema's `.xsd` file(s) into `schemas/` — the file you'll
   point at, plus anything it imports (they must sit alongside each other,
   same as `UCI_MessageDefinitions_v2_5_0.xsd` importing
   `UCI_SecurityMarkings_v2_5_0.xsd` here).
3. Edit exactly two places in `.github/workflows/bake.yml`:
   - the `env:` block (`SCHEMA_PATH`, `SCHEMA_ID`) — informational, read by
     humans and mirrored in the GitLab file;
   - the `with:` block on the `bake` job, which must repeat the same two
     values (GitHub Actions cannot interpolate `env:` into a reusable
     workflow's `with:`, so the two blocks are kept manually in sync — see
     the comment in the file).
4. Push to `main` — the bake runs on both Linux architectures
   (`x86_64`/`arm64`) and uploads the raw and Conan-cache-save tarballs as
   workflow artifacts.
5. Tag a release (`git tag vX.Y.Z && git push origin vX.Y.Z`, tag name must
   start with `v`) to attach those same four tarballs (2 arches x raw +
   Conan) to a GitHub Release.

## Use this template (GitLab)

`.gitlab-ci.yml` is self-contained — it clones the library repo at
`LIBRARY_REF` rather than `include:`-ing the library's own CI config, so it
works on any GitLab instance without mirroring anything but the library
itself.

1. Import this repo into GitLab (or push a copy there).
2. Set `LIBRARY_REPO_URL` to an internal Git mirror of
   `exificient-native-image` if `github.com` egress isn't available from
   your runners — see "Air-gap manifest" below.
3. Set `BAKE_RUNNER_TAGS` to **one** runner tag. GitLab expands a `tags:`
   entry to select a single tag, not a list — if you need multiple
   candidate runners, tag one runner with the value you choose here.
4. Point `BAKE_BUILDER_IMAGE` at the builder image, either pulled from
   `ghcr.io/m4lwar/exificient-builder` or loaded into an internal registry
   (`podman save` / `podman load`, see below).
5. Push. The `bake` job runs `bake.sh` inside the builder image
   (`--network=none`, fully offline once the image and source are local)
   and uploads `bake-out/` as a job artifact. If a project-level Conan
   remote is configured (`CI_SERVER_HOST` + a `conan` binary on the
   runner), the job also uploads the package there.

## Consuming the output

**Via Conan** (recommended — this is how
[`exi-demo`](https://github.com/M4lwar/exi-demo) consumes the baked UCI
package):

```sh
conan cache restore conan-exificient-1.0.0+uci-2.5.0-linux-x86_64.tgz
```
```ini
[requires]
exificient/1.0.0

[options]
exificient/*:baked_schema=uci-2.5.0
```
`baked_schema` is metadata identifying which variant you restored —
setting it doesn't bake anything itself; the baking already happened when
CI ran `bake.sh`.

**Raw tarball** (library + headers, no Conan): grab
`exificient-1.0.0+uci-2.5.0-linux-<arch>.tgz` directly from a release
asset, e.g.:

```
https://github.com/<you>/<your-fork>/releases/download/v1.0.0-uci-2.5.0/exificient-1.0.0+uci-2.5.0-linux-x86_64.tgz
```

## Worked example: uci-2.5.0

This repo's own `schemas/` and workflow files are not placeholders — they
bake for real, and the numbers below are measured from that bake:

| Metric | Value |
|---|---|
| Schema | UCI 2.5.0 (`UCI_MessageDefinitions_v2_5_0.xsd` + imported `UCI_SecurityMarkings_v2_5_0.xsd`, ~8.6 MB combined) |
| Baked artifact (raw tgz, per arch) | ~13.2 MB |
| Baked artifact (Conan cache-save tgz, per arch) | ~13.1 MB |
| Baked `.so` size vs. generic build | +16.3 MB (+56%) |
| `native-image` build time | ~1m 9s |
| Full bake wall time (both Maven phases + native-image + packaging) | ~15-16 min per architecture |
| `exi_create(NULL)` on the baked context | **~4.1 ms** |
| `exi_create(schemaPath)` cold-loading the same schema on a generic build | **~14 s** |

Push this repo unmodified and it reproduces those numbers on both Linux
architectures.

## Air-gap manifest

Nothing in the bake needs `github.com`/`ghcr.io` egress once these three
pieces are staged locally — the container runs with `--network=none`:

1. **Builder image** (GraalVM + a pre-warmed offline Maven repo + Conan):
   `podman save ghcr.io/m4lwar/exificient-builder:1.0.0-x86_64 -o
   builder.tar`, transfer, `podman load -i builder.tar`, or push/pull via
   an internal registry.
2. **Library source** at a pinned ref (default `master`, switch to a
   release tag once the library re-cuts one): an internal Git mirror, or a
   flat source tarball passed to `bake.sh --source`.
3. **This repo** — the schema(s) plus the two CI files. No further
   dependencies; `.gitlab-ci.yml` clones the library itself and never
   reaches out to any other host.

## License

MIT — see `LICENSE`. Third-party notices for `libexificient`'s own
dependencies are in `THIRD_PARTY_NOTICES.txt`.
