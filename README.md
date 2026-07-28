# Sophon Archiver

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Deploy](https://github.com/AJBrownson/sophon-archiver/actions/workflows/deploy.yml/badge.svg)](https://github.com/AJBrownson/sophon-archiver/actions/workflows/deploy.yml)

**Sophon Archiver** is a serverless live broadcast archiving system. A web interface accepts a broadcast URL, dispatches a GitHub Actions workflow to capture the stream via `yt-dlp`, uploads the result to Cloudflare R2, and serves it for in-browser playback or download — all without any infrastructure management or running processes on a personal machine.

## Architecture

```
User ──► Cloudflare Worker (UI + API)
                │
                ├── POST /api/trigger ──► GitHub Dispatches API ──► Actions Runner (yt-dlp) ──► R2 Bucket
                │
                ├── GET /api/recordings ──► R2 Bucket (list)
                │
                └── GET /api/watch/{key} ──► R2 Bucket (stream with Range/seek support)
```

- **Cloudflare Worker** — serves the front-end assets and exposes a REST API for triggering, listing, and streaming recordings
- **GitHub Actions** — a self-contained runner that executes `yt-dlp` to capture the live stream and uploads the segments to R2
- **Cloudflare R2** — object storage for recorded video, with zero egress fees

## Prerequisites

- A [GitHub](https://github.com) account
- A [Cloudflare](https://dash.cloudflare.com) account (free tier sufficient)
- [Node.js](https://nodejs.org/) 18+ and `npm`

## Deployment

### 1. Repository Setup

Create a new GitHub repository (public or private) and push this project to it.

> **Private vs public:** Public repositories receive unlimited free Actions minutes, but workflow run logs are publicly visible. Private repositories are limited to 2,000 free minutes per month (sufficient for occasional capture; a 2-hour stream consumes approximately 120 minutes). Choose based on your privacy requirements and expected usage volume.

Navigate to **Settings → Actions → General → Workflow permissions** and verify Actions are enabled.

### 2. Cloudflare R2 Bucket

1. Log in to the [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Navigate to **R2** and create a new bucket (e.g., `live-archive`)
3. Update `bucket_name` in `wrangler.toml` if you choose a different name

No additional API tokens are required for the Worker — it accesses R2 through a native binding configured in `wrangler.toml`.

### 3. GitHub Actions Secrets

Navigate to **Settings → Secrets and variables → Actions → New repository secret** and add the following credentials. An R2 API token can be created under **R2 → Manage R2 API Tokens**.

| Secret               | Description                                      |
| -------------------- | ------------------------------------------------ |
| `R2_ENDPOINT`        | `https://<account-id>.r2.cloudflarestorage.com`  |
| `R2_ACCESS_KEY_ID`   | R2 API access key                                |
| `R2_SECRET_ACCESS_KEY` | R2 API secret key                              |
| `R2_BUCKET`          | Name of the R2 bucket (e.g., `live-archive`)     |

### 4. Worker Authentication Token

1. Navigate to **GitHub Settings → Developer settings → Fine-grained personal access tokens** and generate a new token
2. Scope it to the repository created in step 1
3. Required permissions: **Contents: Read-only**, **Actions: Read and write**
4. Copy the token value

### 5. Configure Wrangler

Install dependencies and authenticate with Cloudflare:

```bash
npm install
npx wrangler login
```

Set the Worker secrets required to dispatch workflow runs:

```bash
npx wrangler secret put GH_TOKEN    # fine-grained PAT from step 4
npx wrangler secret put GH_OWNER    # GitHub username or organization
npx wrangler secret put GH_REPO     # repository name from step 1
```

### 6. Deploy

```bash
npx wrangler deploy
```

Wrangler outputs a public URL (e.g., `https://sophon-archiver.<subdomain>.workers.dev`).

## Usage

1. Navigate to the Worker URL from any device
2. When a live broadcast is active, paste its URL, optionally provide a title, and click **Start Recording**
3. The tab may be closed — the recording continues on GitHub's infrastructure
4. Return to the Worker URL at any time; completed recordings appear under **Saved Recordings** with **Watch** (in-browser streaming with seek support) and **Download** options

## Configuration Reference

| Secret / Variable | Source                          | Required |
| ----------------- | ------------------------------- | -------- |
| `GH_TOKEN`        | GitHub fine-grained PAT         | Yes      |
| `GH_OWNER`        | GitHub username / organization  | Yes      |
| `GH_REPO`         | GitHub repository name          | Yes      |
| `R2_ENDPOINT`     | Cloudflare R2 dashboard         | Yes      |
| `R2_ACCESS_KEY_ID` | Cloudflare R2 dashboard        | Yes      |
| `R2_SECRET_ACCESS_KEY` | Cloudflare R2 dashboard    | Yes      |
| `R2_BUCKET`       | Chosen bucket name              | Yes      |

## Limitations

- **Execution timeout:** GitHub Actions runners terminate after 6 hours. Streams exceeding this duration require manual splitting, which is not implemented.
- **Mid-stream capture:** Recording begins when the workflow starts, not at the broadcast's origin. For broadcasters with DVR functionality, adding `--live-from-start` to the `yt-dlp` invocation in `.github/workflows/record.yml` enables capture from the beginning of the stream buffer.
- **Single-job status tracking:** The status panel reflects the most recent workflow run. Concurrent recordings are supported at the infrastructure level but are not tracked independently in the UI.

## License

[MIT](LICENSE)