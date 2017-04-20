---
layout: page
title: "Speed"
category: relapse
date: 2016-04-18
order: 6
---

#### Small

Given the following relatively small message which has been randomly populated.

~~~
message Person {
  optional string Name = 1;
  repeated Address Addresses = 2;
  optional string Telephone = 3;
}

message Address {
  optional int64 Number = 1;
  optional string Street = 2;
}
~~~

We get the following benchmarks for the respective expressions listed below.

~~~
(.Name == "David" & .Telephone == "0123456789")				835 ns/op	      0 allocs/op
.Addresses:_[Number == 456, Street == "TheStreet"]			912 ns/op	      0 allocs/op
(@name|@nil) #nil=!(.Name:*) #name=.Name!="David"			339 ns/op	      0 allocs/op
(@empty|@nil) #empty=.Name->eq(length($string),0) #nil=!(.Name:*)	671 ns/op	      0 allocs/op
.Name != "David"							345 ns/op	      0 allocs/op
.Name:->eq(length($string),0)						679 ns/op	      0 allocs/op
!(.Name:*)								256 ns/op	      0 allocs/op
( .Name == "David" | .Telephone == "0123456789")			826 ns/op	      0 allocs/op
.Addresses[*,_:.Number:== 2,_:.Number:== 1]				389 ns/op	      0 allocs/op
~~~

That is millions of matches per second.

#### Recursive

Given the randomly populated recursive message below,

~~~
message SrcTree {
  optional string PackageName = 1;
  repeated SrcTree Imports = 2;
}
~~~

We get the following speed for the recursive expression.

~~~
([PackageName:== "io",*]|[*,Imports._:@main])	      			1751 ns/op	      0 allocs/op
~~~

That is hundreds of thousands of matches per second.

#### Medium

Given the following normal sized message which has been randomly populated.

~~~
message TypewriterPrison {
    optional bytes WineMessenger = 1;
    optional bytes ShoelaceBeer = 2;
    optional PocketRoses PocketRoses = 3;
}

message PocketRoses {
    optional string ScarBusStop = 1;
    optional int64 BadgeShopping = 2;
    optional int64 DaisySled = 3;
    optional int64 SubmarineSaw = 4;
    optional bool SmileLetter = 5;
    optional BullySunrise IconHope = 6;
    optional HopeArch VanPurse = 7;
    repeated string MenuPaperclip = 8;
    repeated string BeetlePoker = 9;
    repeated string WigPride = 10;
    optional DivorceFair DivorceFair = 11;
    repeated fixed32 FlightParachute = 12;
    repeated fixed32 BeerRace = 13;
    repeated bytes LoftQuarry = 14;
    repeated bytes TaxiDivorce = 15;
    repeated fixed32 ElectionButter = 16;
    repeated bytes BriefcaseBaboon = 17;
    optional string MapShark = 18;
    optional bool NetInterlude = 19;
}
~~~

We get the following benchmarks for the respective expressions listed below.

~~~
.PocketRoses.DaisySled == 1						9653 ns/op	      0 allocs/op
.PocketRoses.MapShark *= "a"						9759 ns/op	      0 allocs/op
.PocketRoses.MenuPaperclip: [ _ *= "a", *]				7030 ns/op	      0 allocs/op
.PocketRoses.ScarBusStop *= "a"						6116 ns/op	      0 allocs/op
.PocketRoses.SmileLetter::$bool						6526 ns/op	      0 allocs/op
~~~

That is hundreds of thousands of matches per second.

#### Large

Given [a relatively large message](https://github.com/katydid/testsuite/blob/master/relapse/gen-relapse-tests/puddingmilkshake.proto).

We get the following benchmarks for the respective expressions listed below.

~~~
.FinanceJudo.SaladWorry.SpyCarpenter.BridgePepper:_ *= "a"		14905 ns/op	      0 allocs/op

.FinanceJudo.SaladWorry.SpyCarpenter:
	(.BridgePepper:_ *= "a" & .FountainTarget:_ *= "a") 		15572 ns/op	      0 allocs/op
~~~

That is tens of thousands of matches per second.

### How to reproduce

Benchmarks were run using golang version 1.6 on a 2,7 GHz Intel Core i7 MacBook Pro with 16 GB 1600 MHz DDR3 memory.

This speed was achieved using the [parser/proto.NewProtoNumParser](https://godoc.org/github.com/katydid/katydid/parser/proto#NewProtoNumParser) and the [relapse/protonum.FieldNamesToNumbers](https://godoc.org/github.com/katydid/katydid/relapse/protonum#FieldNamesToNumbers) function.

You can also run the benchmarks yourself:

~~~
git clone https://github.com/katydid/testsuite $GOPATH/src/github.com/katydid/testsuite
(cd $GOPATH/src/github.com/katydid/testsuite && make regenerate)
go test -v -bench=. github.com/katydid/katydid/relapse/auto
~~~

### Memoized vs Compiled

These benchmarks have been run on the compiled automata or auto package.
The mem package contains a memoizing implementation of relapse.
This achieves comparable speeds to the compiled version, given some number of iterations.

[See our comparable benchmarks](../bench/index.html)

Reproducing these benchmarks requires using the b.N argument:

~~~
go test -v -bench=. github.com/katydid/katydid/relapse/mem -args -b.N=10
~~~

The compilation of relapse into an automata can have some state explosions resulting in very long compilations.
This is why the mem package is the default implementation.  It provides comparable speeds and does not have any state explosion problems.

