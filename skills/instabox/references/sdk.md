# instabox TypeScript SDK

Package: `@insforge/instabox-sdk` (docs written against v0.5.0). Full reference: https://instabox.sh/docs/sdk

Runs on **your server** with an API key. Never ship the key to a browser.

```sh
npm install @insforge/instabox-sdk
export INSTABOX_API_KEY=ibk_...
# optional, default https://api.instabox.sh
export INSTABOX_API_URL=https://api.instabox.sh
```

Every static call also accepts `{ apiKey, apiUrl, requestTimeoutMs, operationTimeoutMs, fetch, pollIntervalMs, retryDelayMs }`; options win over env. No key → `InstaboxAuthError` (`missing_api_key`) before any request.

## Create, find, delete

```ts
import { Sandbox } from '@insforge/instabox-sdk'

// template ID, private snapshot name, or snapshot ID (sn_...); default 'agent-browser'
const sbx = await Sandbox.create('universal', {
  name: 'job-1',                     // default sbx-<random>
  metadata: { customer: 'acme' },    // labels for Sandbox.list
  timeoutMs: 30 * 60_000,            // 60 s – 24 h
  onTimeout: 'pause',                // or 'kill'
  autoSuspendMs: 10 * 60_000,        // default 30 min, 0 = never
  vault: false,                      // keep your Vault out (untrusted code / other users)
  envs: { NODE_ENV: 'test' },        // client-side, added to every commands.run
})
console.log(sbx.sandboxId)           // m_...

const again = await Sandbox.connect(id)                    // resumes if paused/stopped
const peek  = await Sandbox.connect(id, { resume: false }) // attach without waking
const list  = await Sandbox.list({ metadata: { customer: 'acme' } })
await sbx.kill()                     // or Sandbox.kill(id); false if already gone
```

Size options (`vcpus`, `memMiB`, `diskGiB`, `rootDiskGiB`) are refused on api.instabox.sh (`400 spec_not_allowed`).

## Lifecycle

| Call | Effect |
|---|---|
| `sbx.pause()` | suspend, memory kept |
| `sbx.resume()` | resume paused, or start stopped |
| `sbx.stop()` | shut down, `/data` kept |
| `sbx.setTimeout(ms, { onTimeout })` | deadline, `0` clears |
| `sbx.setAutoSuspend(idleMs)` | 2 min – 24 h, `0` off |
| `sbx.setMetadata({...})` | replace labels, `{}` clears |
| `sbx.getInfo()` / `sbx.isRunning()` | fresh state |

States: `creating starting running stopping stopped suspending suspended resuming deleting deleted error`.

## Commands

```ts
const r = await sbx.commands.run('python3 -c "print(6*7)"', {
  cwd: '/data/home', timeoutMs: 60_000, envs: { K: 'V' }, stdin: '...',
})
r.stdout; r.stderr; r.exitCode; r.timedOut; r.truncated
```

- Non-zero exit throws `CommandExitError` (same fields). Timeout → `timedOut: true`, exit code 124.
- Output is returned at the end, not streamed; each stream capped at 1 MiB. `timeoutMs` max 600 000. stdin max 1 MiB.
- Background: `const h = await sbx.commands.run('node server.js', { background: true }); await h.kill()`.
- Commands are never retried.

## Files

```ts
await sbx.files.write('/data/home/app.py', 'print("hi")')   // creates parents, atomic
await sbx.files.read('/data/home/app.py')                   // text
await sbx.files.read('img.png', { format: 'bytes' })         // Uint8Array
await sbx.files.list('/data/home')                          // [{ name, path, type: 'file'|'dir' }], not recursive
await sbx.files.exists(path); await sbx.files.remove(path)  // remove is recursive
```

Max 10 MiB per read/write. Relative paths start at `/data/home`. Needs the same scope as commands (`machines:write`).

## SSH credentials

```ts
const ssh = await sbx.sshCredentials(publicKey, { ttlSeconds: 3600 }) // 1 – 86 400
// save ssh.certificate as <key>-cert.pub, add ssh.knownHostsLine to known_hosts, then
// ssh -p <ssh.port> <ssh.username>@<ssh.host>
```

## Desktop (agent-browser template; others install it on first start, ~30 s)

```ts
await sbx.desktop.start({ width: 1280, height: 800 })  // idempotent
await sbx.desktop.open('https://example.com')
const png = await sbx.desktop.screenshot()
await sbx.desktop.click(640, 400, { button: 'right', double: false })
await sbx.desktop.type('hello')
await sbx.desktop.press(['ctrl', 'l'])
await sbx.desktop.scroll({ direction: 'down', amount: 5 })
// also: move, drag([x1,y1],[x2,y2]), cursorPosition(), status()

const { url, expiresAt } = await sbx.desktop.streamUrl({ viewOnly: true, ttlSeconds: 3600 })
await sbx.desktop.revokeStreamUrls() // kills all links + open viewers
```

Stream link works without an instabox account and can go in an `<iframe>`. View-only is enforced server-side. Mint a fresh short-lived link per user request; revoke when access ends.

## Snapshots

```ts
const snap = await sbx.createSnapshot({ name: 'my-setup', rootOnly: false })
const copy = await Sandbox.create(snap.id, { vault: false }) // options are NOT inherited
await Sandbox.listSnapshots(); await Sandbox.getSnapshot(id); await Sandbox.deleteSnapshot(id)
```

## Vault

`sbx.vault()` (what applies), `sbx.pushVault()` (write now), `sbx.disableVault()` (permanent).

## Metrics and usage

```ts
await sbx.getMetrics({ since: new Date(Date.now() - 15 * 60_000) }) // 30 s samples, ≤24 h back
await Sandbox.getUsage(id, { from: new Date('2026-10-01T00:00:00Z') }) // runningSeconds, vcpuSeconds, ...
await Sandbox.usage() // account counts, storage, limits
```

## Errors

| Class | When |
|---|---|
| `InstaboxError` | base: `status`, `code`, `message`, `requestId`, `retryable`, `operationId` |
| `InstaboxAuthError` | 401/403, `missing_api_key`, `insufficient_scope` |
| `InstaboxQuotaError` | 429 `quota_exceeded` |
| `InstaboxTimeoutError` | request/operation too slow |
| `CommandExitError` | non-zero exit |

Other codes: `not_found`, `operation_conflict` (409), `spec_not_allowed`, `is_a_directory`, `network_error`. Reads and idempotent changes retry 3× on 502/503/504/network errors.

## Scopes per call

- `machines:read`: `list`, `getInfo`, `isRunning`, `vault`, `getMetrics`, `getUsage`, `usage`
- `machines:write`: `create`, `connect`, `pause`, `resume`, `stop`, `setTimeout`, `setMetadata`, `disableVault`, `pushVault`, `commands`, `files`, `sshCredentials`, desktop
- `machines:delete`: `kill`
- `images:read` (also needed to create from a snapshot name), `images:write` (`createSnapshot`), `images:delete`

**Security:** a key with `machines:write` can read your Vault via `exec` in machines it creates. For machines that serve other people, always pass `vault: false`.
