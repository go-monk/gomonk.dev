+++
date = '2026-07-13T11:25:50+02:00'
title = 'Default to Go for infrastructure'
+++

Go is not the only language I use. But it's my default language for CLI tools, infrastructure and services. I don't use other languages unless there's a good reason. Here goes why.

SIMPLICITY and CLARITY. There is already enough technological and organizational complexity and confusion. Simpler and clearer systems are easier, cheaper, and more enjoyable to understand, maintain, and operate.

LINGUISTIC STABILITY. The Go team has made a strong compatibility promise, so you don’t need to worry much about programs you write stopping to work as the language evolves. The language and standard library only add; they essentially never break what already worked.

CROSS-COMPILATION to a SINGLE BINARY. Go makes it easy to build (statically linked) executable binaries for different platforms, which simplifies deployment enormously: `GOOS=linux GOARCH=arm64 go build && deploy.sh`

SAFETY and SECURITY. Go is a relatively young language designed with modern safety and security concerns in mind. This is less true of languages created in a much earlier era when the security landscape was very different.

OPTIMIZED for SDLC. Go improves the whole SDLC since it's not just a language but also a toolchain (`go mod tidy`, `go test`, `govulncheck`) and an ecosystem of [packages](https://pkg.go.dev/) (libraries).

And several of these advantages are even more important in the [AI-assisted](https://spf13.com/p/go-the-agentic-language/) development. 