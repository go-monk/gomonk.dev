+++
date = '2026-09-02T08:23:24+02:00'
title = 'Shift Cipher in Go'
+++

A simple way to encipher, or encrypt, some data is by using the so-called shift cipher. We can do this in Go by going through the data byte by byte adding a key to each of the bytes.

```go
func Encipher(plaintext []byte, key byte) []byte {
	ciphertext := make([]byte, len(plaintext))
	for i, b := range plaintext {
		ciphertext[i] = b + key
	}
	return ciphertext
}
```

In Go, bytes are equivalent to 8-bit numbers[^1], thus ranging from 0 to 255. Encrypting using the shift cipher actually means that each byte is shifted by `key` positions, wrapping around at 256. Bytes are often used to represent (encode) alphabet letters, so we are effectively shifting a letter's position in the alphabet.

To decipher we need to do the same but in reverse, i.e. we subtract the key from each byte of the enciphered data.

```go
func Decipher(ciphertext []byte, key byte) []byte {
	return Encipher(ciphertext, -key)
}
```

This way Alice and Bob can exchange data in a somewhat secure manner. If Eve wants to learn what they are talking about she needs to know the encryption algorithm and the key. Let's say she finds out they are using the shift cipher[^2] so she just needs to crack the key. The standard way to do this is called brute forcing, i.e. trying out all possibilities - in our case all possible keys. She also needs to know some bytes from the beginning of the "plaintext" data; this we call a crib. 

```go
func Crack(ciphertext, crib []byte) (key byte, err error) {
	for guess := 0; guess < 256; guess++ {
		result := Decipher(ciphertext[:len(crib)], byte(guess))
		if bytes.Equal(result, crib) {
			return byte(guess), nil
		}
	}
	return 0, errors.New("no key found")
}
```

If we call these functions (from within a main package stored under `./cmd`) it looks like this:

```sh
$ echo HAL | go run ./cmd/encipher
IBM
$ echo IBM | go run ./cmd/decipher
HAL
$ echo hello world | \
  go run ./cmd/encipher -key 10 | \
  go run ./cmd/crack -crib hell
hello world
```

See [shift](https://github.com/jreisinger/pocs/tree/main/crypto/shift) for all the code. Most of the ideas and code come from John Arundel's [book](https://bitfieldconsulting.com/books/crypto) I started to read.

This article is a review of [my older blog post](https://jreisinger.blogspot.com/2024/02/shift-cipher-in-go.html).

[^1]: The `byte` data type is actually an alias for `uint8`.

[^2]: Sometimes also called the Caesar cipher.