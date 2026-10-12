---
title: "Building Trivia Vault: PowerPoint Parsing, Cloud Copies, and a Raspberry Pi Deployment"
aside: true
categories:
  - Raspberry Pi
  - Home Server
feature_image: "/assets/pictures/PS1_1546.jpg"
---

Building a private home for my trivia games, with a FastAPI catalog, PowerPoint question extraction, and copies in Google Drive and Nextcloud:

<!-- This paragraph above controls how much shows in the post description. -->
<!-- more -->

After building Trivia League to run the competition, I wanted a better way to organize the games themselves.

My collection is mostly PowerPoint presentations grouped into folders for trivia nights. I might make eight games for one night, with music or pictures mixed in. The presentations already live on my Mac and in cloud storage, but browsing folders does not tell me much about the questions inside them.

**Trivia Vault** gives that collection a small web interface. I can upload games, group them by night, browse the catalog, and inspect the categories and questions without opening every presentation.

It runs on `JiggleJointPi` alongside the rest of my homelab, at `trivia-vault.jackmazza.xyz`, behind Cloudflare Access.

## Keeping the application small

FastAPI handles the Python side: receiving uploads, validating files, saving metadata, parsing presentations, and coordinating cloud copies. Jinja templates render the pages, with a little JavaScript for upload progress and results.

There is no separate frontend framework or database in this version. The catalog reads JSON metadata stored beside each presentation.

The interface uses the same dark green, bronze, and cream palette as Trivia League. Search, pagination, and a trivia-night filter make a growing collection easier to browse. Upload results have a clear status, with technical details tucked into an expandable section.

## A night can contain several games

The important grouping is the trivia night. Eight presentations can belong to the same night, whether I upload them together or one at a time.

The upload form lets me select an existing night or create a new one. Multiple selected presentations upload sequentially, and later files reuse the night created by the first successful upload. Each file gets its own result.

This is presentation selection rather than recursive folder importing. Accompanying music and PNG files are not imported as a separate collection. Much of the music is already embedded in my presentations, so preserving the original file covers that need for now.

The browser sends each file to `/presentations/upload`. Python does the saving and returns JSON; JavaScript uses that response to update the page. The loading animation shows that a request is running, while the result distinguishes a successful local save from the outcome of the cloud copies.

## Extracting the questions

My usual game has eleven regular category rounds with two questions each. A regular round has two question slides and two answer slides. The bonus round in the middle varies much more.

Some bonuses have several question slides with answers in the slide notes. Others have a single prompt followed by an answer slide. That variation made real examples much more useful than designing a parser around one imaginary template.

The parser reads the XML inside supported PowerPoint files. It preserves category names, extracts question and answer text, and records slide references. It never executes embedded macros.

Music and picture rounds currently keep their categories and slide references. There is no audio recognition or image OCR. Legacy `.ppt` files can be stored, but their contents are not parsed by this implementation.

When a layout does not fit the expected structure, the metadata carries review warnings. The original presentation remains available.

I also moved the category mappings into `app/category_mappings.json`, so I can extend them without changing Python. A collection scan helped populate that list, with typo headings and placeholders left out. Mapping changes affect newly parsed uploads; they do not silently rewrite existing records.

## Files first

The canonical copy lives on an ext4 flash drive mounted at `/mnt/trivia-vault`.

```text
/mnt/trivia-vault/
├── .google-folder-ids.json
└── presentations/
    └── <presentation-id>/
        ├── source.pptm
        ├── metadata.json
        └── cloud_sync.json
```

`metadata.json` describes the game, its trivia night, and extracted content. `cloud_sync.json` tracks the remote-copy results. The hidden Google registry remembers the remote folders associated with trivia nights.

The application needs write access to the storage root as well as the presentations directory because it stages uploads and writes tracking files there. The container runs as UID/GID `10001`; the host directories are provisioned accordingly.

## Google Drive and Nextcloud

After saving locally, the app attempts one-way copies to Google Drive and Nextcloud. Each destination receives the original presentation and its metadata, organized into night and game folders.

A failed remote copy does not undo the local upload. The catalog reports each provider's status and offers a retry. Existing remote files are checked rather than blindly overwritten.

Google permissions took a little adjustment. I wanted the limited `drive.file` scope rather than access to every file in my account. The solution was an app-created Trivia Vault folder that I can still browse in the normal Drive interface. My older collection stays separate.

Nextcloud uses an app password and WebDAV under `Master Trivia/Trivia Vault`.

Copies currently happen during the upload request. The browser uses asynchronous requests, but that does not make the server work a durable background job. A long copy can still encounter a timeout, and the saved status is the last result rather than continuous monitoring of the remote files.

## From GitHub to the Pi

The production branch is `master`. Pull requests run the automated tests on GitHub-hosted runners. Once merged, the workflow builds a `linux/arm64` image, publishes it to GHCR with the commit SHA as its tag, and deploys it through a dedicated Pi runner.

```text
Pull request → tests
                    
Merge to master → tests → ARM64 image → GHCR → Pi runner → Docker Compose
```

The runner uses the labels `self-hosted`, `Linux`, `ARM64`, and `trivia-vault`. It is registered separately from the existing Trivia League runner.

For this application, the deploy job checks out the repository into the runner workspace to copy the Compose file and startup script. The application itself runs from the published container image. The persistent stack files live under `/opt/homelab/stacks/trivia-vault`.

The startup script checks that the expected flash drive is mounted, verifies storage and credential access from inside the container, and waits for Docker's health check. The deployment saves the last healthy image reference in the stack's `.env` file. It does not automatically roll back a failed deployment.

The initial production container reported healthy on the Pi. The local test suite had 27 passing tests at this milestone.

## Cloudflare Tunnel and GitHub sign-in

The tunnel routes `trivia-vault.jackmazza.xyz` to:

```text
http://trivia-vault:8000
```

Both the app and `cloudflared` use the existing `public_web` Docker network. The local troubleshooting port is bound to `127.0.0.1:8089`, so the tunnel uses the Docker service address rather than that loopback port.

I configured a self-hosted Cloudflare Access application for the hostname, using GitHub sign-in and an Allow policy restricted to my account. The email address in the policy identifies the account; it does not mean I need to request an email code. GitHub is the selected login method for this application.

The whole Vault is private. That fits its role as my collection-management tool.

## Local development without touching the cloud

I wanted repeatable commands for coming back to the project later:

```bash
bash scripts/setup_local_dev.sh
bash scripts/start_local_dev.sh
```

The setup script creates `.venv`, installs dependencies, and prepares `data/local-dev/presentations`. The startup script serves the app at `http://127.0.0.1:8000` with reload enabled and cloud copies disabled. A different port can be passed as its first argument.

Tests run separately:

```bash
.venv/bin/python -m pytest
```

The local development guide includes a confirmation-based cleanup command for disposable uploads and a separate, explicit way to test real cloud copies. Local cleanup never deletes remote files.

## What I am leaving out of Restic

I am keeping the presentations out of the homelab Restic job. The intended copies are on the Mac, Pi, Google Drive, and Nextcloud. Stack configuration and secrets under `/opt/homelab` remain within the existing encrypted backup's scope.

The Vault runner is installed under my home directory, outside that backup's source paths, so its installation and work directory are not included either.

This choice has a recovery tradeoff: the Pi's cloud tracking files are not included with the remote presentation copies. Losing them would mean reconnecting the app to existing Google objects or accepting a fresh upload with possible duplicates. This version does not automatically reconstruct those connections. I am accepting that tradeoff for this collection.

## What still needs finishing

The biggest operational follow-up is guarded startup after a reboot. Automatic container restart is currently disabled because I do not want Docker starting against an unmounted flash-drive path. The next step is a boot service that requires the storage mount before starting the stack, followed by a reboot test.

The remaining end-to-end checks are signing in through the public hostname, confirming an unapproved session cannot reach the Vault, and uploading a disposable game through the deployed app to verify both cloud copies. A healthy container proves the process responds; it does not prove those external connections work.

Longer term, a durable background queue would make larger imports less dependent on request timeouts. For now, the foundation is small enough to understand: presentations and JSON on disk, a straightforward Python app, explicit remote copies, and a deployment pipeline that runs on the same Pi as the rest of the homelab.

---

## Implementation Notes

| Component | Configuration |
| --- | --- |
| Application | FastAPI, Jinja templates, JavaScript |
| Production branch | `master` |
| Container platform | `linux/arm64` |
| Canonical storage | `/mnt/trivia-vault` on the flash drive |
| Stack directory | `/opt/homelab/stacks/trivia-vault` |
| Credentials | `/opt/homelab/secrets`, mounted read-only |
| Local health endpoint | `http://127.0.0.1:8089/health` |
| Tunnel origin | `http://trivia-vault:8000` |
| Public hostname | `trivia-vault.jackmazza.xyz` |
| Access | GitHub sign-in restricted to my account |

For the related build, see [Building Trivia League]({% post_url 2026-10-04-Trivia-League-App %}).
