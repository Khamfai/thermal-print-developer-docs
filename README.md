# Thermal-print-developer-docs

A free print service for thermal receipt printers on Android and iOS. Set up your printer once in the app. After that, any other app or web page can print through it: send a job and the app does the printing. There is no license fee, account, watermark, or internet connection involved.

- **Printer languages:** ESC/POS (Epson-compatible, which covers almost every low-cost printer) and StarPRNT (Star Micronics).
- **Connections:** Bluetooth Classic (Android only), Bluetooth LE, and LAN/Wi-Fi (raw TCP, port 9100).
- **Content:** raw printer bytes, text, images, PDF and HTML. Thai and every other non-ASCII script is rendered to an image with an embedded font, so it prints correctly even on printers with no Thai font.
- **App languages:** English and Thai.

> **Status:** phases 1–3 (Android and iOS) are built and covered by automated tests. On-device verification is still in progress; see [the device checklist](docs/superpowers/plans/2026-11-21-phase1-device-checklist.md). StarPRNT has not yet been tested on a real Star printer. Store release is phase 4.

## Ways to send a print job

| Channel | Android | iOS |
|---|:-:|:-:|
| `mdprint://` link from a web page or app, with an optional callback | ✅ | ✅ |
| Intent from another app, with the result returned | ✅ | — |
| Share sheet / "Open in" (image, PDF, text) | ✅ | ✅ |
| System print menu (Chrome, Gallery, …) | ✅ | — |
| Local HTTP server on `127.0.0.1:18800` (opt-in) | ✅ | — |

iOS does not allow background services, so on iOS another app opens this app to print.

**Permission prompts.**
- Android asks before printing for any app or website it hasn't approved yet. Approvals can be revoked in Settings.
- On iOS, links print straight away. To ask every time, turn on **Confirm every job** in Settings.
- Files that are shared or opened from another app always ask first.

## Job format (JSON, version 1)

```json
{
  "v": 1,
  "printer": "optional-printer-id",
  "copies": 1,
  "items": [
    { "type": "text", "text": "ร้านกาแฟ / Coffee shop", "align": "center", "size": 2, "bold": true },
    { "type": "text", "text": "Latte            65.00" },
    { "type": "image", "data": "<base64 PNG or JPEG>" },
    { "type": "pdf", "data": "<base64 PDF>", "trim": true },
    { "type": "html", "html": "<p>Thank you</p>" },
    { "type": "raw", "data": "<base64 printer bytes>" },
    { "type": "feed", "lines": 3 },
    { "type": "drawer" },
    { "type": "cut" }
  ]
}
```

**Top-level fields**

| Field | Values |
|---|---|
| `v` | Must be `1`. |
| `printer` | Optional. Leave it out to use the default printer. |
| `copies` | 1–10. Default 1. |

**Item types**

| `type` | Fields |
|---|---|
| `text` | `text` (required), `align` (`left` \| `center` \| `right`), `size` (1 or 2), `bold` (boolean) |
| `image` | `data`: base64 PNG or JPEG, scaled to the paper width |
| `pdf` | `data`: base64 PDF. `trim: true` cuts each page just below its last printed line. |
| `html` | `html`: an HTML string, rendered offline. The renderer runs no scripts and makes no network requests; only `data:` URLs load. |
| `raw` | `data`: base64 bytes, sent to the printer unchanged |
| `feed` | `lines`: 1–20 |
| `drawer` | Opens the cash drawer. |
| `cut` | Cuts the paper. |

**Limits**

| What | Limit |
|---|---|
| Total job data | 5 MiB |
| Items per job | 500 |
| PDF pages | 20 |
| Rendered height | 20,000 dot rows |
| HTTP request body | 7 MiB |
| Opened images on iOS | 50 megapixels |

## Integration

### Link (web pages and apps, Android and iOS)

```
mdprint://print?job=<base64url JSON job>&callback=<url>
mdprint://print?raw=<base64url printer bytes>&callback=<url>
```

```js
const job = { v: 1, items: [{ type: 'text', text: 'Hello' }, { type: 'cut' }] };
const json = new TextEncoder().encode(JSON.stringify(job));
const b64url = btoa(String.fromCharCode(...json))
  .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
location.href = `mdprint://print?job=${b64url}&callback=${encodeURIComponent(location.href)}`;
```

**Encoding.** Pass exactly one of `job` or `raw`, encoded as base64url. Padding is optional.

**Callback.** After printing, the app opens `callback` with `status=ok|error|denied|cancelled` added to its query string, plus `error_code` when the job failed.
- On Android, the callback must be an `http(s)` URL.
- On iOS, the callback can be an `http(s)` URL or another app's own URL scheme, such as `mypos://printed`. System schemes such as `tel:`, `sms:`, `shortcuts:` and `javascript:` are ignored.

### Android Intent (apps)

```kotlin
val launcher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { r ->
    val status = r.data?.getStringExtra("status")        // "ok" | "error" | "denied"
    val errorCode = r.data?.getStringExtra("error_code") // when status is "error"
}

val job = """{"v":1,"items":[{"type":"text","text":"Hello"},{"type":"cut"}]}"""
launcher.launch(
    Intent("com.mounoydev.print.action.PRINT")
        .setPackage("com.mounoydev.print")
        .putExtra("job", job), // or .putExtra("raw", byteArray)
)
```

The first request from your app shows an approval prompt.

### Local HTTP server (Android, opt-in)

Turn it on in **Settings → Print from web pages**. A web page can then print silently, without switching apps:

```js
const res = await fetch('http://127.0.0.1:18800/print', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(job),
});
const result = await res.json(); // { status: 'ok' } or { status: 'error', error_code, message }
```

**Permission prompts.**
- Chrome 142 and later asks once for local network access.
- The app asks once per website, using a notification with Allow and Deny.

`GET /status` returns `{"api":1,"app":"thermal-print"}`.

**HTTP status codes**

| HTTP | Meaning |
|---|---|
| 200 | `ok` |
| 400 | `invalid_job` |
| 403 | The site was not approved |
| 408 | `timeout`: the body arrived too slowly |
| 409 | `no_printer` |
| 413 | `too_large` |
| 429 | `rate_limited` |
| 502 | `printer_unreachable` or `write_failed` |
| 503 | `busy` |

### Error codes (all channels)

| Code | Meaning |
|---|---|
| `invalid_job` | The job is malformed. |
| `too_large` | The job exceeds a limit. |
| `no_printer` | No printer is set up. |
| `printer_unreachable` | The app could not connect to the printer. |
| `write_failed` | The connection dropped while printing. |
| `denied` | The user did not allow the request. |


## Privacy

M Thermal Print stores everything on your device and sends nothing to any server. See the [privacy policy](privacy.md) ([ภาษาไทย](privacy-th.md)).
