# vite-plugin-byteguard

Vite plugin that encodes JavaScript bundles into binary format, preventing casual source code exposure.

**How it works:** At build time, entry JS chunks are encoded into a custom binary format (`.bin`). At runtime, a small inline loader fetches the binary, decodes it, and executes it via a Blob URL.

**Performance:** the cost sits at load time. Before the entry runs, the page
fetches the `.bin` and decodes it — and inflates it when `compress: 'gzip'` is
set — so startup grows with bundle size, algorithm, compression, and device.
After decoding, the entry runs as ordinary JavaScript. No benchmark is published
for either phase; measure startup on the devices you ship to.

## Install

```bash
npm install vite-plugin-byteguard -D
```

## Usage

```js
// vite.config.ts
import { defineConfig } from 'vite'
import byteguard from 'vite-plugin-byteguard'

export default defineConfig({
  plugins: [
    byteguard()
  ]
})
```

### Options

```ts
byteguard({
  // Encoding algorithm: 'xor' (default) or 'aes-gcm'
  algorithm: 'xor',

  // Key size in bytes (default: 32)
  keySize: 32,

  // Glob patterns to exclude from encoding
  exclude: [],

  // Encoded file extension (default: 'bin')
  extension: 'bin',

  // Also encode worker bundles (default: false) — see Workers below
  workers: false
})
```

## What Gets Encoded

| Asset Type | Encoded? | Protection |
|------------|----------|------------|
| Entry JS chunks | ✅ Binary `.bin` | Unreadable on disk |
| Dynamic import chunks | ❌ `.js` as-is | Use with obfuscator |
| Web Workers (default) | ❌ `.js` as-is | Use with obfuscator |
| Web Workers (`workers` option) | ✅ Binary `.bin` | Start with `loadWorker` — see [Workers](#workers) |
| CSS / Assets | ❌ Untouched | N/A |

> [!TIP]
> Pair with [rollup-plugin-obfuscator](https://www.npmjs.com/package/rollup-plugin-obfuscator) for full coverage — obfuscate all JS, then byteguard encodes the main bundle.

## Algorithms

| Algorithm | Speed | Protection | Async Required |
|-----------|-------|------------|----------------|
| `xor` | ⚡ fastest | casual protection | No |
| `aes-gcm` | fast | strong encryption | Yes (WebCrypto) |

> [!WARNING]
> Use `aes-gcm` whenever `compress: 'gzip'` is on. A gzip payload starts with
> the known bytes `1f 8b 08`, and XOR against a known prefix hands over that
> many bytes of the key — with a repeating key, all of it within the first
> key-length of the file. The combination is allowed, and it is still weak.

## Keeping the Key Out of the Build

By default the key travels in the `.bin`, which stops casual browsing and
nothing more. `keySource: 'native'` leaves it out entirely: the page supplies
it at runtime, typically from a native bridge on mobile.

```js
byteguard({
  algorithm: 'aes-gcm',
  keySource: 'native',
  key: myKeyBytes,        // the same bytes the device will return
  compress: 'gzip'
})
```

```html
<!-- index.html, before the generated loader runs -->
<script>
  window.__byteguardKey = await MyNativePlugin.getKey()
</script>
```

The provider may hold bytes, an `ArrayBuffer`, a function returning either, or
a promise. Without it the loader throws and nothing is fetched or executed.
`fallback: 'header'` puts the key back in the file for simulator and debug
builds, where no bridge exists.

## Workers

Workers are left alone by default. `workers: true` encodes emitted worker
bundles too; `workers: ['**/*.worker-*.js']` names them by pattern. A pattern is
matched against every emitted `.js` asset and non-entry chunk, so keep it narrow
enough to miss dynamic import chunks, which must stay loadable as JavaScript.
`exclude` applies to workers as well.

The plugin renames each encoded worker to `.bin` and replaces its
bundle-relative path (for example `assets/sim.worker-<hash>.js`) wherever that
string appears in the emitted chunks and workers. A URL Vite writes in another
form is not rewritten and keeps pointing at the deleted `.js`; one such case,
with `base: './'`, is tracked in
[#10](https://github.com/Blosharper/byteguard/issues/10).

An encoded worker cannot be started by the constructor — the browser would
fetch the `.bin` and parse it as JavaScript. Start it from the page instead:

```js
import { loadWorker } from 'vite-plugin-byteguard/runtime'

// before
const worker = new Worker(new URL('./sim.worker.js', import.meta.url), { type: 'module' })

// after
import workerUrl from './sim.worker.js?worker&url'
const worker = await loadWorker(workerUrl, { type: 'module' })
```

`?worker&url` is what gives `loadWorker` a URL that Vite has actually built a
worker bundle for; `new URL('./x.worker.js', import.meta.url)` on its own is an
asset reference, not a worker build. `loadWorker` takes the key the same way
the entry loader did, so it needs no configuration of its own.

> [!NOTE]
> Worker code is encoded as it stands. The `import.meta.url` and dynamic
> `import()` rewrites the plugin applies to the entry resolve against
> `document.baseURI`, and a worker has no `document` — so a worker that itself
> relies on either is not a candidate for encoding. Leave it out of `workers`.

## CSP Requirements

The loader is an inline `<script>` that starts the decoded entry as a module
from a `blob:` URL, so `script-src` has to let both run. The example allows the
inline loader with `'unsafe-inline'`:

```html
<meta http-equiv="Content-Security-Policy"
  content="script-src 'self' 'unsafe-inline' blob:;" />
```

- The loader reads the `.bin` with `fetch()`. A policy that restricts
  `connect-src`, or sets `default-src` without it, has to allow the origin that
  serves the `.bin` (usually `'self'`).
- `loadWorker` starts an encoded worker from a `blob:` URL. If the policy sets
  `worker-src` or `child-src`, include `blob:` there; otherwise `script-src`
  applies.
- The Blob URL module path is used when the replaced `<script>` tag has
  `type="module"`, which is how Vite emits its entry. For a tag without it, the
  loader runs the code with `new Function`, which also needs `'unsafe-eval'`.

## How It Works

```
Build Time:
  index.js → [obfuscator, optional] → [gzip, optional] → [XOR / AES-GCM encode] → index.bin
  import('./chunk.js') → import(new URL('assets/chunk.js', document.baseURI).href)

Runtime:
  index.html
    └─ <script> (inline loader)
        ├─ fetch('./assets/index.bin')
        ├─ XOR / AES-GCM decode (then inflate, if gzipped)
        └─ Blob URL → <script type="module" src="blob:...">
            └─ runs as an ordinary module
```

Workers load as normal `.js` files unless `workers` is set. The plugin automatically rewrites `import.meta.url` and dynamic `import()` paths so that asset references resolve correctly from the Blob URL execution context. This ensures compatibility with WKWebView (iOS/Capacitor) and other strict module environments.

> [!TIP]
> If you use `rollup-plugin-obfuscator` with `stringArray` encoding, add `reservedStrings: ['\\.js$']` to prevent import paths from being encoded into the string array.

## Binary Format

```
Offset  Size    Description
0       4       Magic "BGRD" (0x42 0x47 0x52 0x44)
4       1       Version (0x01)
5       1       Algorithm (0x01=XOR, 0x02=AES-GCM)
6       2       Key length (uint16 LE) — 0 when the key is not in the file
8       N       Key bytes (absent when the length is 0)
--- AES-GCM only ---
8+N     1       IV length (12)
9+N     12      IV bytes
--- End ---
varies  rest    Encoded payload
```

A compressed payload needs a flag byte the layout above has nowhere to put, so
those files are written as version 2: the same fields with `Flags` inserted at
offset 6, moving everything after it along by one. A missing key does **not**
bump the version — every field is length-prefixed, so `KeyLen = 0` shortens the
header unambiguously. Full description:
[`byteguard`](https://www.npmjs.com/package/byteguard#binary-format).

## Limitations

- The inline loader script is visible in HTML (contains the decoding logic)
- A determined attacker can intercept the decoded JS in memory via DevTools
- This is **casual protection** — it prevents string searching, source browsing, and automated scraping of your JS bundles
- Workers remain as plain JS unless `workers` is set, and encoding them means calling `loadWorker` instead of the constructor
- Dynamic import chunks remain as plain `.js` files, but their import paths are automatically resolved

## Use Cases

- **Capacitor/Cordova** mobile apps — prevent APK/IPA extraction → source reading
- **Electron** apps — complement bytenode for web-facing parts
- **Hybrid apps** where JS source should not be trivially accessible

## License

MIT
