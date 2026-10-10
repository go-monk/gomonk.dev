+++
date = '2015-10-28'
author = 'Alan Donovan'
title = 'spinner'
+++

One `go` statement keeps a terminal spinner turning while `main` grinds through a slow computation. Nobody stops the spinner; when `main` returns, the program ends and takes it along.

```go
func main() {
	go spinner(100 * time.Millisecond)
	const n = 45
	fibN := fib(n) // slow
	fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {
	for {
		for _, r := range `-\|/` {
			fmt.Printf("\r%c", r)
			time.Sleep(delay)
		}
	}
}

func fib(x int) int {
	if x < 2 {
		return x
	}
	return fib(x-1) + fib(x-2)
}
```

Source: [gopl.io/ch8/spinner](https://github.com/adonovan/gopl.io/blob/master/ch8/spinner/main.go) from *The Go Programming Language* by Alan Donovan and Brian Kernighan, licensed [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
