---
layout: page
title: "Test Suite"
category: tests
date: 2017-10-09
order: 7
---

#### Test Suite

The test suite can be found [here](https://github.com/katydid/testsuite) and is a language agnostic test suite.

The idea is that katydid can be implemented in multiple programming languages. Currently an implementation is available in [Go](https://github.com/katydid/katydid) and one in [Haskell](https://github.com/katydid/katydid-haskell).
We only need one set of tests, that should pass for any implementation.
This should help to keep implementations consistent and help with creating any new implementation.

Technically each of the two implementations consist of three implementations of Relapse's core algorithm, so currently there are six users of the test suite.

The test suite is simply a bunch of folders and files that can be traversed and opened in any programming language.

The test suite contains some Go code, only to help to generate tests for multiple serialization formats. The output is just a bunch of files and folders.
There should be no need to run this, except when adding more tests, since all the output files and folders are already committed in the repository.
The reason for generating files and folders is that katydid is encoding agnostic, so to make sure that the parsers are implemented consistently, we might want the same relapse test for multiple serialization formats.

This Go code for test suite generation should not be confused with the Go tests for the Go implementations of Relapse.
The Go implementations of Relapse all expect the test suite to located adjacent to katydid in the `GOPATH` and the files are read like in any other language's implementation, without using any Go code from the test suite repository.
