# Firefox ArrayBuffer Blob Video Reproduction

This repository contains a minimal reproduction for a Firefox video-loading issue involving an MP4 fetched as an `ArrayBuffer` and converted into a Blob URL.

## Summary

In Firefox Release, assigning a Blob URL created from a fetched `ArrayBuffer` to a `<video>` element can leave the video stuck in the loading state indefinitely.

The same page works as expected in Google Chrome.

Loading the original URL directly also works in Firefox Release. Creating the Blob URL from `Response.blob()` instead of rebuilding the Blob from an `ArrayBuffer` works as well.

## Reproduction

The failing path is:

```js
const buffer = await fetch(src).then((response) => response.arrayBuffer());
const blob = new Blob([buffer], { type: "video/mp4" });
const blobUrl = URL.createObjectURL(blob);

video.src = blobUrl;
```

To reproduce:

1. Clone this repository.
2. Serve the repository over HTTP and open `index.html` in Firefox Release.
3. Observe that the video remains in the loading state and does not begin playback.
4. Open the same page in Chrome and observe that the video loads normally.

Do not open `index.html` directly through a `file://` URL, because browser security and fetch behavior may differ from an HTTP origin.

## Control cases

### Direct URL

Assigning the remote URL directly works:

```js
video.src = src;
```

### Blob returned by `fetch()`

Using the Blob produced by the Fetch API also works:

```js
const blobUrl = await fetch(src)
  .then((response) => response.blob())
  .then((blob) => URL.createObjectURL(blob));

video.src = blobUrl;
```

### Blob reconstructed from an `ArrayBuffer`

The problem occurs with this otherwise equivalent path:

```js
const buffer = await fetch(src).then((response) => response.arrayBuffer());
const blob = new Blob([buffer], { type: "video/mp4" });
video.src = URL.createObjectURL(blob);
```

Although the resulting Blob contains the same response bytes, Firefox may use a different internal Blob or stream implementation for these construction paths.

## Tested Firefox builds

| Result | Version | Build ID | Operating system | Source revision |
| --- | --- | --- | --- | --- |
| Affected | Firefox 152.0.6 | `20260713164047` | macOS | [`27b462b22705a8860f7ab0d33aa5b4b658ae5932`](https://github.com/mozilla-firefox/firefox/commit/27b462b22705a8860f7ab0d33aa5b4b658ae5932) |

Chrome is not affected in the tested environment.

## Test environment

- Model: MacBook Pro (16-inch, November 2024)
- Chip: Apple M4 Pro
- Memory: 24 GB
- Operating system: macOS 15.6 (`24G84`)

## Expected behavior

The Blob URL should load and play in the `<video>` element, regardless of whether the Blob was returned by `Response.blob()` or constructed from the response's `ArrayBuffer`.

## Actual behavior

In the affected Firefox Release build, the video remains in the loading state. The same media URL and bytes work through the control paths described above.

## Related investigation

[Mozilla Bug 2052818](https://bugzilla.mozilla.org/show_bug.cgi?id=2052818) describes a very similar symptom involving MP4 data fetched as an `ArrayBuffer` and reconstructed as a Blob. It has not yet been confirmed that the two reports have the same root cause.

## Notes

- The reproduction currently fetches `https://lorem.video/720p`, so it requires network access and depends on that service remaining available and returning a CORS-enabled MP4 response.
- For a permanent browser test case, the remote video should eventually be replaced with a small MP4 checked into this repository.
- Results may depend on the returned MP4 file and its container metadata, not only on the JavaScript construction path.

## License

This reproduction code may be used freely for debugging, testing, and reporting this browser issue.
