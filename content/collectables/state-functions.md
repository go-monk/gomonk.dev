+++
date = '2011-06-13'
author = 'Rob Pike'
title = 'State functions'
+++

A lexer state is a function that returns the next state. No state enum, no giant switch; the machine is a loop that keeps calling whatever it was handed. From Rob Pike's lexer for Go templates.

```go
// stateFn represents the state of the scanner as a function that returns the next state.
type stateFn func(*lexer) stateFn
```

```go
// nextItem returns the next item from the input.
// Called by the parser, not in the lexing goroutine.
func (l *lexer) nextItem() item {
	l.item = item{itemEOF, l.pos, "EOF", l.startLine}
	state := lexText
	if l.insideAction {
		state = lexInsideAction
	}
	for {
		state = state(l)
		if state == nil {
			return l.item
		}
	}
}
```

Source: [text/template/parse/lex.go](https://github.com/golang/go/blob/master/src/text/template/parse/lex.go), explained in Rob Pike's talk [Lexical Scanning in Go](https://go.dev/talks/2011/lex.slide)
