<p align="center">
  <a href="https://mitrafsnap.github.io"><img src="https://img.shields.io/badge/Project%20page-mitrafsnap.github.io-cc3333" alt="Project page"></a>
  <a href="https://github.com/mitrafsnap/demo"><img src="https://img.shields.io/badge/Live%20demo-mitrafsnap%2Fdemo-cc3333" alt="Live demo"></a>
  <img src="https://img.shields.io/badge/License-Apache--2.0-1c39bb" alt="License Apache-2.0">
</p>

# MitraFSnap builder

> Part of **MitraFSnap**, Build Less, Pull Less, Restart Never: Fast Container Image
> Distribution and In-Place Updates. The paper is under review.
> [Project page, videos and results](https://mitrafsnap.github.io)

MitraFSnap updates the data mounted in a running container without replacing the container.
It consists of this builder, a registry and a containerd snapshotter. The builder packages
each model, configuration or asset of an image as an **allotment** that is built and cached
independently, so an update rebuilds only the allotments it changes, and independent
allotments build in parallel.

Every allotment is compressed as eStargz or zstd:chunked, and the builder writes its chunk
index (TOC) during the build. The image is therefore lazy loadable without a separate
conversion pass, and the chunk indexes later let the snapshotter tell which bytes of an
updated allotment actually changed.

| Repository | Role in MitraFSnap |
|---|---|
| **builder** (this repository) | builds independent, lazy loadable allotments and their chunk indexes |
| [registry](https://github.com/mitrafsnap/registry) | turns a partition selection into prefetch hints over the complete image |
| [snapshotter](https://github.com/mitrafsnap/snapshotter) | lazy pulling and refresh of running containers |
| [demo](https://github.com/mitrafsnap/demo) | live demo that compares Docker, eStargz, 2DFS and MitraFSnap |
| [evaluation](https://github.com/mitrafsnap/evaluation) | build, pull and update benchmarks |

## Build and install

The builder is a single Go binary called `tdfs` and needs Go 1.25 or later.

```sh
git clone https://github.com/mitrafsnap/builder.git
cd builder
go build -o tdfs ./internal/tdfs.go
sudo install tdfs /usr/local/bin/
```

## Usage

An image consists of a base OCI image and a field of allotments. `2dfs.json` places each file
of the build context in a row and a column of the field.

```json
{
  "allotments": [
    { "src": "./detector.onnx", "dst": "/models/detector.onnx", "row": 0, "col": 0 },
    { "src": "./segmenter.onnx", "dst": "/models/segmenter.onnx", "row": 0, "col": 1 },
    { "src": "./config.json", "dst": "/models/config.json", "row": 0, "col": 2 }
  ]
}
```

Build the image with lazy loadable allotments and push it to the MitraFSnap registry.

```sh
tdfs build --enable-stargz --use-zstd --force-http -f 2dfs.json \
  ubuntu:24.04 registry.example:5000/app:v1
tdfs image push --force-http registry.example:5000/app:v1
```

When a file changes, the same two commands rebuild and push only its allotment. A client
selects a partition of the field with a tag suffix `--row1.col1.row2.col2`, which contains every
allotment with a row between `row1` and `row2` and a column between `col1` and `col2`. For
example, `app:v1--0.0.0.0` selects only the detector. Several suffixes can be chained, and
their union is selected.

| `tdfs build` flag | Meaning |
|---|---|
| `--enable-stargz` | build allotments as eStargz layers with a chunk index |
| `--use-zstd` | use zstd:chunked instead of gzip |
| `--stargz-chunk-size N` | chunk size in bytes, 4 MiB by default |
| `--stargz-compression-level N` | gzip level for eStargz, from 1 (fastest) to 9 (smallest) |
| `--zstd-compression-level N` | zstd level, from 1 (fastest) to 4 (best) |
| `--compression-level N` | gzip level for allotments without lazy loading |

The global flag `--home-dir`, or the variable `TDFS_HOME`, moves the cache of the builder.
`tdfs image export` writes a partition as an OCI archive that `docker load` accepts.

## Relation to the 2DFS builder

This repository is a fork of the [2DFS builder](https://github.com/2DFS/2dfs-builder), which
introduced independently built allotments. MitraFSnap adds eStargz and zstd:chunked
allotments with chunk indexes generated during the build, configurable compression,
compressed allotment sizes, base images without an index or with v1 manifests, and a
relocatable cache. The upstream documentation is in [UPSTREAM.md](UPSTREAM.md).

## License

Apache License 2.0, as upstream. See [LICENSE](LICENSE).
