# WebGPU Player decoders

The decoder projects of [WebGPU Player](https://github.com/alchemyyy/WebGPU-Player).
Each folder with a `decoder.json` is one project, which builds one decoder package: a WebAssembly module whose custom sections carry its WGSL decoder or audio manifest and its self-test vectors, and which a host loads into the engine at runtime.
`atmos/` holds the Dolby Atmos object audio sources the `eac3` and `truehd` projects compile in, `include/` the audio decoder interface for C modules, and `licenses/` the upstream license texts.

| Project | Decodes |
| --- | --- |
| `av1` | AV1 Main, High, and Professional at 8 to 12 bits, on the GPU, with libaom's decoder as its parser |
| `hevc` | HEVC Main, Main 10, and the range extensions, on the GPU |
| `jpeg2000` | JPEG 2000, and Digital Cinema X'Y'Z' as BT.709, on the GPU |
| `mpeg2` | MPEG-1 and MPEG-2 Video, on the GPU |
| `mpeg4` | MPEG-4 Part 2, H.263, and MS-MPEG4 versions 1 to 3, on the GPU |
| `vc1` | VC-1 Advanced and WMV3, on the GPU |
| `uncompressed` | Uncompressed planar packets, on the GPU; the template of a WGSL-only decoder |
| `eac3` | AC-3 and E-AC-3, with the Dolby Atmos objects of joint object coding, from FFmpeg |
| `truehd` | MLP and TrueHD, with the objects of TrueHD Dolby Atmos, from FFmpeg |
| `wma` | WMA 1, WMA 2, WMA Pro, WMA Lossless, and WMA Voice, from FFmpeg |
| `dts` | DTS core, DTS-HD High Resolution, and DTS-HD Master Audio, from libdcadec |

The engine checks this repository out as its `wgpud/` submodule, and builds and tests the projects with its own tools.
Its book describes them, their build, and the package format, in `docs/src/decoders.md` and `docs/src/decoder-packages.md` of the engine repository.

## Build

The projects build only inside an engine checkout: the engine's `scripts/decoder-packages.mjs` packs every project and loads each package with the engine's own loader, and the engine's `vendor/` holds the FFmpeg, dcadec, and libaom sources that `rules.mk` pins.
They need GNU Make, Git, a POSIX shell (Git Bash on Windows), Node.js 24, Emscripten 4.0.13, rustup, CMake, Perl, and GNU patch, as the engine's `docs/src/setup.md` lists.
On Windows they need long path support too: the FFmpeg and cargo trees pass 260 characters in a nested checkout, so enable `LongPathsEnabled` and Git's `core.longpaths`, and GNU Make and the MSVC linker that cargo uses must accept long paths.
From the engine root:

```sh
git submodule update --init wgpud
make -C wgpud sources      # Fetch the pinned FFmpeg, dcadec, and libaom, once
npm run decoders           # Build every project into bin/wgpud/<project>.wasm
npm run decoders -- hevc   # Build one project
make -C wgpud clean
```

A project's tests, in its `test/`, run in the engine's Vitest against the package in `bin/wgpud/`: `npx vitest --watch=false wgpud/<project>`.

## Releases

`.github/workflows/release.yml` builds every package in a checkout of the engine, with this repository at its `wgpud/`, and publishes them as a GitHub release.
Run it by hand on `main` (Actions > Release > Run workflow) with a version such as `0.2.0`, which becomes the tag `v0.2.0`.
Without a version it is a dry run, which keeps the release files as the run's artifact.
Its `engine-ref` input names the engine branch, tag, or commit whose build steps it runs.

A release carries:

- `<project>.wasm`, every package;
- the license texts in `licenses/`;
- the corresponding source of the LGPL packages, which `make -C wgpud source-archives` writes: `ffmpeg-<commit>.tar.gz`, `dcadec-<commit>.tar.gz`, and `webgpu-player-decoders-<revision>.tar.gz`, this repository at the released commit;
- `SHA256SUMS`, the SHA-256 of every file.

The engine pins one release, file by file, in its `tools/pinned-releases.json`, and its `build.sh` and `sandbox.sh` install the pinned packages into `bin/wgpud/`.
`node scripts/pinned-releases.mjs --pin decoders <tag>` moves the pin to another release.

## License

Our own sources are MIT; see [LICENSE](LICENSE).
FFmpeg and libdcadec are LGPL-2.1-or-later, and so are the `eac3`, `truehd`, `wma`, and `dts` packages, which contain them, and `truehd/truehd_presentation.c` and `truehd/truehd_substream.c` with their headers, which port FFmpeg's MLP decoder.
libaom, which the `av1` package contains, is BSD-2-Clause.
The HRIR table in `atmos/atmos_hrir.c`, which the `eac3` and `truehd` packages contain, comes from the MIT KEMAR measurements of Bill Gardner and Keith Martin (MIT Media Lab, 1994), which whoever distributes those packages must cite; the file's header does.
The streams in `truehd/vectors/` that come from the truehd crate's test assets keep that crate's Apache-2.0 license; see its `LICENSE` and `PROVENANCE.txt`.
