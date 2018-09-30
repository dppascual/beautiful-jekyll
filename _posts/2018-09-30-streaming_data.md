---
layout: post
title: Streaming data in Go
gh-repo: dppascual/go-cookbook/dataIO
tags: [data, streaming, golang]
---

<head>
<meta charset = "UTF-8">
<link href="https://fonts.googleapis.com/css?family=EB+Garamond" rel="stylesheet">
<link rel="stylesheet" type="text/css" href="../css/styles.css" />
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>

# IO

Input and output operations are provided by a set of primitives that model data as a stream of bytes. In Go, such stream of data is represented as a **slice of bytes([]byte**) that can be accessed for reading or writing.

### The *io.Reader* interface

The *io.Reader* interface is composed of a single method that lets programmers implement code that reads data, from an arbitrary source, and transfers it into the provided slice of bytes.

{% highlight golang linenos %}
type Reader interface {
    Read(p []byte)(n int, err error)
}
{% endhighlight %}

For a type to behave as a reader, it must implement the method *Read* from interface *io.Reader*.

As a guideline, implementation rules of the *Read()* method given on [io.Reader](https://golang.org/pkg/io/#Reader) are as follows:

**Read reads up to len(p) bytes and transfers them into p. It returns the number of bytes read ( 0 <= n <= len(p)) and any error encountered.**

**When *Read* encounters an error or end-of-file condition after successfully reading n > 0 bytes, it returns the number of bytes read. It may return the (non-nil) error from the same call or return the error (and n == 0) from a subsequent call**. An instance of this general case is that a Reader returning a non-zero number of bytes at the end of the input stream may return either err == EOF or err == nil. The next Read should return 0, EOF.

Callers should always process the n > 0 bytes returned before considering the error err. Doing so correctly handles I/O errors that happen after reading some bytes and also both of the allowed EOF behaviors.

**Implementations of *Read* are discouraged from returning a zero byte count with a nil error, except when len(p) == 0**. Callers should treat a return of 0 and nil as indicating that nothing happened; in particular it does not indicate EOF.

#### Streaming data from readers

The way of streaming data from a reader is by using a method *Read*. It is designed to be called within a loop where, by each iteration, a chunk of data is read from the source and transfered it into buffer p.

An example of how a function call *Read* should be implemented is as shown below. A slice of 10 bytes length is used as a buffer `make([]byte, 10)` where the chunk of data is sent to and processed before getting the next chunk. The buffer is purposefully smaller than the length of the source in order to demostrate how to properly stream chunks of data from a source that is larger than the buffer.

{% highlight golang linenos %}
package main

import (
	"fmt"
	"io"
	"os"
	"strings"
)

func main() {
	reader := strings.NewReader("Split a string into chunks of the buffer length")
	buf := make([]byte, 10)
	for {
		n, err := reader.Read(buf)
		if err != nil {
			fmt.Println(string(buf[:n]))
			if err == io.EOF {
				break
			}
			fmt.Println(err)
			os.Exit(1)
		}
		fmt.Println(string(buf[:n]))
	}
}
{% endhighlight %}

#### A custom io.Reader

In this section is shown how to implement a custom IO reader.

{% highlight golang linenos %}
package main

import (
	"errors"
	"fmt"
	"io"
	"os"
	"unicode/utf8"
)

type asciiReader struct {
	src []byte
	cur int
	err error
}

// newASCIIReader is used to take under control the src field when it is empty. A new asciiReader should be created by using this method.
func newASCIIReader(src string) *asciiReader {
	if len(src) == 0 {
		return &asciiReader{
			src: make([]byte, 0),
			err: io.EOF,
		}
	}

	return &asciiReader{
		src: []byte(src),
	}
}

func (a *asciiReader) Read(p []byte) (int, error) {
	var bytesASCII int

	if len(p) == 0 {
		return 0, nil
	}
	if a.err != nil {
		return 0, a.err
	}

	for bytesASCII < len(p) {
		word, sizeWord := utf8.DecodeRune(a.src[a.cur:])

		if word == utf8.RuneError {
			a.err = errors.New("An error was gotten trying to report an Unicode character")
			return bytesASCII, nil
		}

		a.cur += sizeWord

		if sizeWord == 1 {
			p[bytesASCII] = byte(word)
			bytesASCII += sizeWord
		}

		if len(a.src[a.cur:]) == 0 {
			a.err = io.EOF
			return bytesASCII, nil
		}
	}
	return bytesASCII, nil
}

func main() {
	input := newASCIIReader("A new reader to filter out non-ASCII characters 的")
	buf := make([]byte, 10)
	for {
		size, err := input.Read(buf)
		if err != nil {
			if size > 0 {
				fmt.Println(string(buf[:size]))
			}
			if err == io.EOF {
				break
			}
			fmt.Println(err)
			os.Exit(1)
		}
		fmt.Println(string(buf[:size]))
	}
}
{% endhighlight %}

#### Chaining Readers


### The *io.Writer* interface

The *io.Writer* interface is composed of a single method that is designed to read data, from a buffer `[]bytes`, and transfers it into a specified target resource.

{% highlight golang linenos %}
type Writer interface {
    Write(p []byte)(n int, err error)
}
{% endhighlight %}

For a type to behave as a writer, it must implement the method *Write* from interface *io.Writer*. 

As a guideline, implementation rules of the *Write()* method given on [io.Writer](https://golang.org/pkg/io/#Writer) are as follows:

*Write* reads data from the buffer `p` and write it to the underlying data stream. It should return the number of bytes written and any error encountered that caused the write to stop early. *Write* must return a non-nil error if it returns n < len(p) and not modify the slice data, even temporarily.

