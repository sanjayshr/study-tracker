# Prepare Cloudflare Pages publishing

You can finish the OS and hardware setup without a Cloudflare account or token. This guide prepares the tools; project creation and deployment commands are for later, after a complete local dashboard preview and a visibility decision.

## Supported tools and our pins

The project pins **Node.js 24.19.0** in `.nvmrc` and **Wrangler 4.147.0** in `package.json`; `package-lock.json` pins its dependency tree. Node 24 is an LTS line. The Wrangler package requires Node >=22, but the minimum version alone does not establish a supported production installation. [Node release status](https://nodejs.org/en/about/previous-releases).

Cloudflare supports Current, Active LTS, and Maintenance LTS Node versions, and recommends a local Wrangler installation. Its installation page states Linux support requires “glib 2.35”; the upstream runtime clarifies the dependency as **glibc >=2.35**. Windows 11 and macOS 13.5+ are listed. [Wrangler installation requirements](https://developers.cloudflare.com/workers/wrangler/install-and-update/).

The `workerd` runtime lists Linux ARM64 support and requires the ARM CRC instruction extension. OS architecture, glibc, CPU features, and actual command execution must all be checked. This is why a 64-bit CPU alone is insufficient. [Runtime requirements](https://github.com/cloudflare/workerd#running-workerd).

The pins were checked on this repository's Ubuntu x86_64 workspace. They have **not** been verified on your Raspberry Pi. Read [the verification record](verification.md) for the exact boundary.

## Install Node on the 64-bit Pi

**Pi:** First check `dpkg --print-architecture` reports `arm64`, `getconf GNU_LIBC_VERSION` reports at least 2.35, and `/proc/cpuinfo` includes `crc32` in its Features lines:

```sh
dpkg --print-architecture
getconf GNU_LIBC_VERSION
cat /proc/cpuinfo
```

If these checks fail, use the laptop fallback below. Do not patch system glibc or compile a runtime on the 1 GB Pi to force installation.

Download the pinned official ARM64 Node distribution and verify its archive against Node's published checksum file:

```sh
mkdir -p ~/Downloads/study-tracker-node
cd ~/Downloads/study-tracker-node
curl --fail --location --remote-name https://nodejs.org/dist/v24.19.0/node-v24.19.0-linux-arm64.tar.xz
curl --fail --location --remote-name https://nodejs.org/dist/v24.19.0/SHASUMS256.txt
grep ' node-v24.19.0-linux-arm64.tar.xz$' SHASUMS256.txt > node-checksum.txt
sha256sum -c node-checksum.txt
```

Continue only when the checksum reports `OK`. The checksum checks the download against a manifest retrieved over HTTPS; it is not independent signature verification. [Official Node release files](https://nodejs.org/dist/v24.19.0/).

```sh
mkdir -p ~/.local/opt
tar -xJf node-v24.19.0-linux-arm64.tar.xz -C ~/.local/opt
export PATH="$HOME/.local/opt/node-v24.19.0-linux-arm64/bin:$PATH"
node --version
npm --version
```

Add that `export PATH=...` line once to `~/.profile` using `nano ~/.profile`, so it applies to future login sessions. A future systemd service will need an explicit executable path or PATH; systemd does not load your shell profile automatically. An existing Node version manager can instead install and select the version in `.nvmrc`.

## Install and check the local Wrangler

**Pi, from the repository root:**

```sh
cd ~/projects/study-tracker
npm ci
WRANGLER_SEND_METRICS=false npm run wrangler -- --version
WRANGLER_SEND_METRICS=false npm run wrangler -- pages deploy --help
./node_modules/.bin/workerd --version
```

Expect Wrangler 4.147.0. These commands do not deploy a website or need a token. Use the locally installed CLI rather than a global install or a bare `npx` invocation that might download a different release.

If npm reports blocked dependency install scripts, inspect the named packages; Wrangler uses native tooling such as esbuild and workerd. Check the installed npm version's script approval mechanism rather than globally allowing every package. The checked workspace emitted script-policy warnings but the CLI and runtime version checks still passed.

Passing these checks establishes basic startup, not a successful ARM64 production upload or acceptable memory use. If you get an unsupported architecture, GLIBC error, illegal instruction, missing executable, or out-of-memory failure, retain the error and use the laptop fallback. Do not enable automatic publishing yet.

## Laptop fallback

Keep tracking, SQLite, tone playback, and dashboard generation on the Pi. Put Node 24.19.0 and this repository's pinned Wrangler on a supported laptop. Windows 11 can use the official Node installer; macOS 13.5+ can use its official installer; a supported Linux laptop can use the official distribution for its architecture. [Node downloads](https://nodejs.org/en/download).

Once the application produces a complete dashboard, transfer its entire `dist/` directory over the local network to the laptop. For example, on a macOS/Linux laptop in its repository checkout:

```sh
mkdir -p dist
rsync -av --delete student@study-pi.local:~/projects/study-tracker/dist/ ./dist/
```

This example mirrors into a dedicated generated-output directory, removing stale output files there. On Windows, use an SFTP client to replace that dedicated output directory with the complete current Pi `dist/`. Keep it outside Git history. Do not transfer the database, environment file, or unrelated private files with the dashboard.

Run the same local Wrangler commands from the laptop. Tracking remains on the Pi and the website remains on Cloudflare Pages. This fallback is a manual publication workflow unless we later explicitly design another worker; it does not satisfy automatic publication after every ending tap by itself.

## Cloudflare account and unattended token

Create a Cloudflare account when you are ready to publish. A custom domain is optional for a Pages dashboard. Obtain the account ID from your Cloudflare account's dashboard.

Create a custom API token with **Account → Cloudflare Pages → Edit**, restricted to the intended account. Avoid the global API key and unrelated DNS, Workers, or account-administration permissions. This permission is account-scoped; do not describe it as a project-only token. [Cloudflare token instructions](https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/#generate-an-api-token).

The future publisher will read `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` from its process environment. For a service, keep them in a protected environment file outside the checkout and load it using systemd's `EnvironmentFile`; we will provide that service file when the application exists. Never put a token in a command argument, source file, generated asset, browser JavaScript, screenshot, or log. Do not paste tokens into chat.

## Later: create and publish the production project

**Do not run this section during OS preparation.** First show the finished local preview and decide whether real study history should be public or private. A public GitHub repository does not authorise publishing personal dashboard data.

With the two environment variables set securely, create a Direct Upload Pages project once. The proposed explicit name is `study-tracker`; choose another name if unavailable. The production branch is `main`:

```sh
npm run wrangler -- pages project create study-tracker --production-branch main
```

Later, with a complete `dist/` and confirmed visibility, a noninteractive production deployment uses:

```sh
CI=true WRANGLER_SEND_METRICS=false npm run wrangler -- pages deploy dist --project-name study-tracker --branch main
```

The project must already exist; the command must not rely on a prompt or cached branch selection. The future Python publisher will invoke this CLI in the background. [Pages command reference](https://developers.cloudflare.com/workers/wrangler/commands/pages/) and [Direct Upload workflow](https://developers.cloudflare.com/pages/get-started/direct-upload/).

An ordinary Pages site is accessible to anyone with its URL. Private access needs a verified Cloudflare Access setup covering the production hostname, deployment/preview URLs, and static history assets before any real history is uploaded. A hidden URL or browser password is insufficient. Access configuration is still a decision to investigate; this document does not claim protection is already configured.

Each deployment must carry the complete dashboard and retained history. It is a snapshot: new events appear online only after a successful upload. Existing publication remains available with the Pi off or offline. The application will preserve pending updates, serialize deployments, and retry with bounded backoff; those behaviours are not implemented in this preparation repository yet.

## Limits to design around

The current Free-plan Pages limits include 20,000 files per site and 25 MiB per asset. Keep historical dashboard output below both limits rather than uploading SQLite. The limits page lists 500 monthly builds and one concurrent Free-plan build; these hosted-build figures should not be assumed to define every Direct Upload API quota. Respect upload/API errors and rate limits, coalesce pending updates, and avoid a retry storm. [Pages limits](https://developers.cloudflare.com/pages/platform/limits/).
