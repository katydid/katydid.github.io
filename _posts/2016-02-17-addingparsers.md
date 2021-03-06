---
layout: page
title: "Extendable"
category: parser
date: 2016-02-17
order: 2
---

Current implementations of parsers include:

  * [Protocol Buffers](https://developers.google.com/protocol-buffers/)
  * [Json](https://json.org/)
  * [Reflected structures in Go](https://golang.org/pkg/reflect)
  * XML for the dinosaurs and dragons amongst us.

Implementing a parser for a new serialization format or any other parsable text or bytes can be done by implementing the following interface.

{% highlight go %}
type Interface interface {
    Next() error
    IsLeaf() bool
    Up()
    Down()
    Value
}

type Value interface {
    Double() (float64, error)
    Int() (int64, error)
    Uint() (uint64, error)
    Bool() (bool, error)
    String() (string, error)
    Bytes() ([]byte, error)
}
{% endhighlight %}

This interface allows for the implementation of a streaming parser.
A parser that lazily parses the input as the methods are called and only parses the input once, without backtracking.
Exercising the parser can be done with the [debug.Walk](https://godoc.org/github.com/katydid/katydid/parser/debug#Walk) function.  The Walk function also returns some debugging output which should be useful in the development of your parser.

Your parser should also be able to handle skipping of some of the input.  This happens when the Walk function returns before encountering an `io.EOF`.  The [debug.RandomWalk](https://godoc.org/github.com/katydid/katydid/parser/debug#RandomWalk) function is useful for testing this type of robustness in your parser.
