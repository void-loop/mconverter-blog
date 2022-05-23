---
title: Using the Web Share Target API in File Converter PWA
description: Allowing for a new, easier way to choose files – previously inaccessible for web apps.
image: mconverter_share_android_static.webp
image_alt: MConverter in Android's Share Sheet
date_added: 2022-03-16
date_updated: 2022-03-16
---

*This article originally appeared on our Medium blog.*

The Web Share Target API, part of Chromium’s Project Fugu to expand the web capabilities, has opened a new entry point for returning visitors of the MConverter PWA. Instead of having to first open the PWA and then choose files through the boring file picker, users can directly share a file for converting from another app, such as a file manager. See the demo (or [try it yourself](https://mconverter.eu)):

<iframe src="https://www.youtube-nocookie.com/embed/TyClSRvqQcU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

*Once installed (or added to the home screen), the MConverter PWA shows up as a native app*

## Increased productivity and more returning visitors

Being able to convert files from any app is much more convenient and it feels more native. Users are also exposed to the PWA from another place in addition to the home screen: the share sheet. This has led to more returning visitors, with **nearly a quarter of all daily conversions getting initiated through the share menu**, thanks to the Share Target API!

## How to implement it?

Adding support is relatively straightforward by following [this article on web.dev](https://web.dev/web-share-target/). In essence, you need to announce support for receiving shares in your PWA’s `manifest.json`:

    {
        "name": "MConverter",
        ...
        "share_target": {
            "action": "/?homescreen=1&share-target=1",
            "method": "POST",
            "enctype": "multipart/form-data",
            "params": {
                "files": [
                    {
                    "name": "file",
                    "accept": ["image/png", ".png"]
                    }
                ]
            }
        }
    }

❗ **Note:** In contrast to the `<input type="file">`’s accept attribute, here it is very important to include *both* the MIME type and the file extension. Otherwise, by adding only a file extension, you may run into a nasty issue on Chrome for Android where the PWA appears in the share sheet, but it *sometimes* cannot receive the shared file. Instead, the service worker’s `event.request.formData()` throws the following error: `Uncaught (in promise) TypeError: Failed to fetch.`

Next, your service worker needs to process the shared file. In the `fetch` event handler, add the following snippet:

    //service-worker.js:

    self.addEventListener('fetch', (event) => {
        const url = new URL(event.request.url);
        if (event.request.method === 'POST' && url.pathname === '/' && url.searchParams.has('share-target')) {
            event.respondWith(Response.redirect('/?receiving-file-share=1'));

            event.waitUntil(async function () {
                const client = await self.clients.get(event.resultingClientId);
                const data = await event.request.formData();
                const files = data.get('file');
                client.postMessage({ files });
            }());
            return;
        }

        ...

    });

The shared file will be made accessible via `postMessage()` to your website’s “regular” JavaScript code, where you can intercept and handle it the same way as a file from `<input type="file">`:

    //script in index.html:

    navigator.serviceWorker.addEventListener('message', function (e) {
        if (searchParams.has('receiving-file-share')) {
            console.log(e.data.files); //contains the file(s)
        }
    });

## But can you share multiple files at once?

Yes! With this little tweak I found from a helpful [answer](https://stackoverflow.com/a/61872441/3955094) by Jeff Posnick, who is part of Google’s DevRel team, we can process many files from a single share. How convenient!

    const files = data.getAll('file');

This will return all files in an array.

## Next up: “Open With…” menu integration

Having quicker access to the converter PWA from other apps on Android is great, but what about desktop platforms?

The [File Handling API](https://github.com/WICG/file-handling/blob/main/explainer.md) is also part of Chrome’s Project Fugu to bring more capabilities to the web. It will allow an installed PWA to register as a potential handler for a file type.

![“Open With…” context menu in File Explorer on Windows 10](open_with_menu.webp)

In other words, the PWA will appear in the “Open With” menu of file managers on Windows, Mac, Linux and Chrome OS.

Even though it is currently only behind a developer flag, MConverter already has a (mostly complete) implementation for it. So, we are just waiting for the File Handling API to ship officially, in order to make the functionality available without needing to manually flip the switch in chrome://flags.

---

![Animation of MConverter appearing in Android's share sheet](mconverter_share_android.webp)