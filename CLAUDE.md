# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Containerized `@peertube/peertube-runner` for remote execution of transcoding jobs in Kubernetes. Includes utility scripts for PeerTube caption management and DeepL translation.

## Docker Images

Two image variants targeting different PeerTube versions:

| Tag               | PeerTube | Runner  | Base            | Dockerfile        |
|-------------------|----------|---------|-----------------|-------------------|
| `v632`            | v6.3.2   | v0.0.21 | Debian Bookworm | `Dockerfile.v632` |
| `v730` (`latest`) | v7.3.0   | v0.2.0  | Debian Bookworm | `Dockerfile.v730` |

### Build Commands

```bash
# PeerTube 6.3.2 variant
docker buildx build --platform linux/amd64 -f Dockerfile.v632 -t owntube/peertube-runner:v632 .

# PeerTube 7.3.0 variant (latest)
docker buildx build --platform linux/amd64 -f Dockerfile.v730 -t owntube/peertube-runner:v730 .
```

### Test Commands

```bash
# Test v632
docker run -it --rm owntube/peertube-runner:v632 npx peertube-runner --help

# Test v730
docker run -it --rm owntube/peertube-runner:v730 npx peertube-runner --help
```

## CI/CD

GitHub Actions workflow (`.github/workflows/docker-image-ci.yml`) runs on push to `main`:
- Builds both image variants
- Tests each with `--help` command
- Pushes to Docker Hub as `owntube/peertube-runner:{v632,v730,latest}`

Requires Docker Hub secrets: `DOCKER_HUB_USERNAME`, `DOCKER_HUB_TOKEN`

## Utility Scripts (`utils/`)

Python scripts for PeerTube caption management. Setup:

```bash
cd utils/
python3 -m venv venv
source venv/bin/activate
pip install click requests deepl sh
```

### Scripts

| Script                      | Purpose                                  |
|-----------------------------|------------------------------------------|
| `issue-auth-token.py`       | Get PeerTube API auth token              |
| `build-video-inventory.py`  | Build video inventory JSON from PeerTube |
| `slow-jobs-scheduling.py`   | Schedule transcription jobs at slow pace |
| `archive-video-metadata.py` | Archive metadata/captions to Git repo    |
| `convert-vtt-to-srt.py`     | Convert WebVTT to SRT (for DeepL)        |
| `convert-srt-to-vtt.py`     | Convert SRT back to WebVTT               |
| `deepl-translate-srt.py`    | Translate SRT captions via DeepL API     |

### Caption Translation Workflow

```bash
# 1. Convert VTT to SRT
python3 convert-vtt-to-srt.py data/video.vtt data/video.srt

# 2. Translate with DeepL
export DEEPL_API_KEY=your-key
python3 deepl-translate-srt.py data/video.srt EN-US data/video_en.srt --auth_key $DEEPL_API_KEY

# 3. Convert back to VTT
python3 convert-srt-to-vtt.py data/video_en.srt data/video_en.vtt
```

## Kubernetes Deployment

Requires `ReadWriteMany` StorageClass for IPC socket sharing between containers. Three PVCs needed:
- `peertube-runner-local` (~100Mi) - IPC sockets
- `peertube-runner-config` (~100Mi) - config.toml files
- `peertube-runner-cache` (~10Gi) - transcoding temp files

See README.md for full Kubernetes manifests and runner registration commands.

## Adding New PeerTube Versions

When adding support for a new PeerTube version:

1. **Replace oldest variant** - Maintain only 2 image variants; replace the older one rather than adding a third
2. **Dockerfile naming** - Use `Dockerfile.v{major}{minor}{patch}` format (e.g., `Dockerfile.v730`)
3. **Check runner compatibility** - Match runner version to PeerTube release (see [runner changelog](https://github.com/Chocobozzz/PeerTube/blob/develop/apps/peertube-runner/CHANGELOG.md))
4. **Node.js requirements** - Runner v0.1.0+ requires Node.js 20
5. **Pin package versions** - Always pin `whisper-ctranslate2` and other packages to specific versions for supply-chain security (e.g., `pipx install whisper-ctranslate2==0.5.6`)
6. **Update `latest` tag** - Point to newest variant in CI workflow
7. **Update all docs** - README.md, this file, and CI workflow

Resources:
- [PeerTube releases](https://github.com/Chocobozzz/PeerTube/releases)
- [Docker Hub tags](https://hub.docker.com/r/chocobozzz/peertube/tags) - check available base images
- [@peertube/peertube-runner npm](https://www.npmjs.com/package/@peertube/peertube-runner)
