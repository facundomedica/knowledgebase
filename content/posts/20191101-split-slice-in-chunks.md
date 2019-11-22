---
title: "Split slice in chunks"
author: "Facundo Medica"
type: "post"
date: 2019-11-01T19:55:15-03:00
subtitle: ""
image: ""
tags: ["snippets","go"]
draft: false
---

After benchmarking other solutions, this came out to be the most efficient.

[Go Playground](https://play.golang.org/p/eNMOIlUFIRI)

<!--more-->

```go
func split(buf []string, lim int) [][]string {
	var chunk []string
	chunks := make([][]string, 0, len(buf)/lim+1)
	for len(buf) >= lim {
		chunk, buf = buf[:lim], buf[lim:]
		chunks = append(chunks, chunk)
	}
	if len(buf) > 0 {
		chunks = append(chunks, buf[:len(buf)])
	}
	return chunks
}
```