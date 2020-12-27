# statigz

<!--[![Build Status](https://github.com/vearutop/statigz/workflows/test/badge.svg)](https://github.com/vearutop/statigz/actions?query=branch%3Amaster+workflow%3Atest)
[![Coverage Status](https://codecov.io/gh/vearutop/statigz/branch/master/graph/badge.svg)](https://codecov.io/gh/vearutop/statigz)-->
[![GoDevDoc](https://img.shields.io/badge/dev-doc-00ADD8?logo=go)](https://pkg.go.dev/github.com/vearutop/statigz)
[![Time Tracker](https://wakatime.com/badge/github/vearutop/statigz.svg)](https://wakatime.com/badge/github/vearutop/statigz)
![Code lines](https://sloc.xyz/github/vearutop/statigz/?category=code)
![Comments](https://sloc.xyz/github/vearutop/statigz/?category=comments)

`statigz` serves pre-compressed embedded files with http in Go 1.16 and later.

## Why?

Since version 1.16 Go provides [standard way](https://tip.golang.org/pkg/embed/) to embed static assets. This API has
advantages over previous solutions:

* assets are processed during build, so there is no need for manual generation step,
* embedded data does not need to be kept in residential memory (as opposed to previous solutions that kept data in
  regular byte slices).

A common case for embedding is to serve static assets of a web application. In order to save bandwidth and improve
latency, those assets are often served compressed. Compression concerns are out of `embed` responsibilities, yet they
are quite important. Previous solutions (for example [`vfsgen`](https://github.com/shurcooL/vfsgen)
with [`httpgzip`](https://github.com/shurcooL/httpgzip)) can optimize performance by storing compressed assets and
serving them directly to capable user agents. This library implements such functionality for embedded file systems.

## Example

```go
package main

import (
	"embed"
	"log"
	"net/http"

	"github.com/vearutop/statigz"
	"github.com/vearutop/statigz/brotli"
)

// Declare your embedded assets.

//go:embed static/*
var st embed.FS

func main() {
	// Plug static assets handler to your server or router.
	err := http.ListenAndServe(":80", statigz.FileServer(st, brotli.AddEncoding))
	if err != nil {
		log.Fatal(err)
	}
}
```

## Usage

Behavior is based on [nginx gzip static module](http://nginx.org/en/docs/http/ngx_http_gzip_static_module.html) and
[`github.com/lpar/gzipped`](https://github.com/lpar/gzipped).

Static assets have to be manually compressed with additional file extension, e.g. `bundle.js` would
become `bundle.js.gz` (compressed with gzip) or `index.html` would become `index.html.br` (compressed with brotli).

> **_NOTE:_** Although [`brotli`](https://github.com/google/brotli) has better compression than `gzip` and already
> has wide support in browsers, it has limitations for non-https servers,
> see [this](https://bugs.chromium.org/p/chromium/issues/detail?id=452335)
> and [this](https://bugzilla.mozilla.org/show_bug.cgi?id=1218924).

> **_NOTE:_** `brotli` support is optional. Using `brotli` adds about 260 KB to binary size, that's why it is moved to
> a separate package.

> **_NOTE:_** [`zopfli`](https://github.com/google/zopfli) provides better compression than `gzip` while being
> backwards compatible with it.

Upon request server checks if there is a compressed file matching `Accept-Encoding` and serves it directly.

If user agent does not support available compressed data, server uses an uncompressed file if it is available (
e.g. `bundle.js`). If uncompressed file is not available, then server would decompress a compressed file into response.

Responses have `ETag` headers (64-bit FNV-1 hash of file contents) to enable caching. Responses that are not dynamically
decompressed are served with [`http.ServeContent`](https://golang.org/pkg/net/http/#ServeContent) for ranges support.
