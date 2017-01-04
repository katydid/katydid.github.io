---
layout: default
title: "Katydid"
---

## Katydid

<img src="{{ site.url }}/logo.png" width="300" style="float:right">

[**Katydid**](https://github.com/katydid/katydid) is a toolkit for trees.

[![GoDoc](https://godoc.org/github.com/katydid/katydid?status.svg)](https://godoc.org/github.com/katydid/katydid) [![Build Status](https://travis-ci.org/katydid/katydid.svg?branch=master)](https://travis-ci.org/katydid/katydid) Version 0.2 Beta.

Currently there are three tools in the katydid toolkit:

  * **Relapse**: 
    - a regular expression type language for trees (any implemented parser)
    - [**matches up to 1000000s of records per second**](./relapse/relapsespeed.html)
  * A collection of **parsers** which are easily extendable:
    - protobuf
    - json
    - xml
    - reflected go structures
    - [add your own](./parser/addingparsers.html)
  * A collection of **encoders** which are useful for dynamic transcoding from any implemented parser:
    - protobuf
    - json
    - xml
    - reflected go structures

#### What are trees?

A tree is a structure, record, class that does not contain any loops.

Katydid has implemented parsers for multiple types of trees:

  * [Protocol Buffers](https://developers.google.com/protocol-buffers/)
  * [Json](http://json.org/)
  * [Reflected structures in Go](http://golang.org/pkg/reflect)
  * and even XML for the dinosaurs and dragons.
  * Easy to add your own. Simply implement a [parser](./parser/addingparsers.html)

Relapse can validate these trees, since they have implemented Parsers.

Encoders can take parsers and encode (or transcode) them into other types of trees.

#### What is Relapse?

Relapse is a validation language.
Regular expressions are used as validators for strings.
RelaxNG, DTD and XSchema are validation languages for XML.
JsonSchema is a validation language for JSON.
Relapse can validate any tree that has an implemented parser.

[Relapse Tour](http://katydid.github.io/tour)

[Relapse Playground](http://katydid.github.io/play)

#### What is a Katydid?

A Katydid is the common name for a leaf insect.
<iframe width="300" height="169" src="https://www.youtube.com/embed/SvjSP2xYZm8" frameborder="0" allowfullscreen></iframe>

#### External Tools

  * [Translate RelaxNG](https://github.com/katydid/relaxng) to Relapse
  * [Translate JsonSchema](https://github.com/katydid/jsonschema) to Relapse [Work in Progress]

#### Possible Futures

  * Capturing (like Regular Expression Capturing, but for trees)
  * Investigate viability of usage as a type system.
  * More serialization formats: CapnProto?, MsgPack, Thrift, Bson, Yaml, Toml ...
  * Remove the one dependency on gogoprotobuf from the core Katydid repository.
