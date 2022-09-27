# run-a-script
[![SemVer](https://img.shields.io/badge/version-1.0.1-brightgreen.svg)](http://semver.org)
[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](http://www.gnu.org/licenses/gpl-3.0)

[Download from Firefox Addons](https://addons.mozilla.org/en-US/firefox/addon/run-a-script/)

A Firefox extension that allows you to define exactly one JS script and inject it into every web page you visit. It uses Firefox' [userScripts](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/userScripts) feature which sandboxes JS code before execution. You should be safe from even the most malicious of actors out there. For at least some convenience, jQuery is injected as well and usable from your script. The code of the extension is deliberately minimal, you are encouraged to review it for security before running the extension. Please report all problems you find and share your snippets in the [issues](https://github.com/MIvanchev/run-a-script/issues).

run-a-script is meant for people with trust issues. There are plenty of extensions out there that you could easily re-implement with a small JS snippet instead of installing. Also useful with [uBlock Origin](https://github.com/gorhill/uBlock) for defeating aggressive sites.

The plugin supports:
* sandboxed injection of 1 script + jQuery into all pages, you need to refresh the opened ones
* enabling / disabling the script

The plugin does NOT support:
* Android – but working on it
* Chrome – the whole idea of this extension is to be secure but that's irrelevant if your browser is a black box
* saving to and loading from a file – saving would be useful but currently not possible due to a [Firefox bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1292701) 

![The GUI of run-a-script](https://raw.githubusercontent.com/MIvanchev/run-a-script/master/screenshot.png)

## Example

Here's a simple script that does useful redirects:

```
console.log("The execution of the custom code is beginning.");

const GIPHY_REDIRECT_ONLY_GIFS = false;
const GFYCAT_PREFER_GIFS = false;

// The following redirection schemes are supported.
//
// 1. Unconditional redirect
// 2. Redirect based on a simple URL condition
// 3. Complex redirect based on the content of the page
//

var redirects = {
    "www.google.com": ["www.startpage.com", (url) => !url.pathname.startsWith("/maps")],
    "www.youtube.com": "yewtu.be",
    "www.reddit.com": ["old.reddit.com", (url) => !url.pathname.startsWith("/gallery/")],
    "imgur.com": redirectToImgurMedia,
    "gfycat.com": redirectToGfycatMedia,
    "giphy.com": redirectToGiphyMedia,
    "media1.giphy.com": redirectToGiphyMedia,
    "media2.giphy.com": redirectToGiphyMedia,
    "media3.giphy.com": redirectToGiphyMedia,
    "media4.giphy.com": redirectToGiphyMedia,
    "tenor.com": redirectToTenorMedia,
    "twitter.com": "nitter.net",
    "open.spotify.com": openSpotifySongWithInvidious,
    "yewtu.be": handleSpotifyRedirect
}

var url = document.URL;
var {
    host,
    protocol,
    pathname,
    search
} = new URL(url);

function redirectToImgurMedia() {
    if (pathname !== "/")
        redirectFromMetaTagContent("twitter:image", "og:video");
}

function redirectToGfycatMedia() {
    if (pathname !== "/") {
        $.get(url, function(data) {
            var vidUrl = null;
            var imgUrl = null;

            var regexList = [
                /<source[^>]*?\s+src="(https:\/\/giant\.gfycat\.com.+?)"/,
                /<source[^>]*?\s+src="(https:\/\/thums\.gfycat\.com.+?)"/,
                /<source[^>]*?\s+src="(https:\/\/.+?\.gfycat\.com.+?)"/
            ];
            for (regex of regexList) {
                var match = data.match(regex);
                if (match) {
                    vidUrl = match[1];
                    break;
                }
            }
            var match = data.match(/<img[^>]+?actual-gif-image/);
            if (match) {
                match = data.slice(match.index).match(/src="(.+?)"/);
                if (match)
                    imgUrl = match[1];
            }

            var mediaUrl = null;
            if (vidUrl && imgUrl)
                mediaUrl = GFYCAT_PREFER_GIFS ? imgUrl : vidUrl;
            else if (vidUrl)
                mediaUrl = vidUrl;
            else if (imgUrl)
                mediaUrl = imgUrl;

            if (mediaUrl)
                goToUrl(mediaUrl);
        });
    }
}

function redirectToGiphyMedia() {
    if (pathname.startsWith("/gifs/") ||
        pathname.startsWith("/clips/") && !GIPHY_REDIRECT_ONLY_GIFS) {

        mediaId = pathname.match(/-([a-zA-Z0-9]+)$/);
        if (mediaId)
            window.location.replace(`https://i.giphy.com/media/${mediaId[1]}/giphy.gif`);
    } else if (pathname.startsWith("/media/")) {
        if (!search.includes('&ct=v') || !GIPHY_REDIRECT_ONLY_GIFS) {
            mediaId = pathname.match(/^\/media\/([a-zA-Z0-9]+)\//);
            if (mediaId)
                goToUrl(`https://i.giphy.com/media/${mediaId[1]}/giphy.gif`);
        }
    }
}

function redirectToTenorMedia() {
    if (pathname.startsWith("/view/"))
        redirectFromMetaTagContent("twitter:image");
}

function openSpotifySongWithInvidious() {
    if (pathname.startsWith("/track/")) {
        $.get(url, function(data) {
            var tags = getMetaTags(data);
            var searchTerm = null;
            if (tags.get("og:type") == "music.song") {
                title = tags.get("og:title", "");
                artist = tags.get("og:description", "").replace(/ ·.*/, "");
                if (title != "" && artist != "") {
                    searchTerm = encodeURIComponent(`${title} by ${artist}`);
                    goToUrl(`https://yewtu.be/search/?q=${searchTerm}&from-spotify=1`);
                }
            }
        });
    }
}

function handleSpotifyRedirect() {
    if (search.includes("from-spotify=1")) {
        window.stop();
        console.log("Handling Spotify redirect...");
        $.get(url, function(data) {
            match = data.match(/\s(?:href)="(\/watch\?v=.+?)"/);
            if (match)
                goToUrl(`https://yewtu.be${match[1]}`);
        });
    }
}

function getMetaTags(data) {
    var result = new Map();
    var meta_regex = /<meta(?=\s+)/g;
    while ((match = meta_regex.exec(data)) !== null) {
        var end_pos = meta_regex.lastIndex;
        var substr = data.slice(end_pos);
        var match = substr.match(/\s(?:name|property)="(.*?)"/);
        if (match) {
            name = match[1];
            match = substr.match(/\s(?:content)="(.*?)"/);
            if (match)
                result.set(name, match[1]);
        }
    }

    return result;
}

function redirectFromMetaTagContent() {
    var args = arguments
    $.get(url, function(data) {
        var tags = getMetaTags(data);
        for (arg of args) {
            if (tags.has(arg)) {
                goToUrl(tags.get(arg));
                break;
            }
        }
    });
}

function goToUrl(url) {
    window.stop();
    window.location.replace(url);
}

var redirect = redirects[host];

if (redirect) {
    if (typeof redirect === "object") {
        if (redirect[1](new URL(document.URL))) {
            redirect = redirect[0];
            window.location.replace(url.replace(
                `${protocol}//${host}`,
                `${protocol}//${redirect}`
            ));
        }
    } else if (typeof redirect === "string") {
        window.location.replace(url.replace(
            `${protocol}//${host}`,
            `${protocol}//${redirect}`
        ));
    } else
        redirect();
}

console.log("The execution of the custom code is finishing.");
```

## License

Copyright © 2022-present Mihail Ivanchev.

Distributed under the GNU General Public License, Version 3 (GNU GPLv3).

The doge icon was sourced from https://icon-library.com/icon/doge-icon-21.html and subsequently modified.

jQuery is distributed under [its respective license](https://jquery.org/license/).

