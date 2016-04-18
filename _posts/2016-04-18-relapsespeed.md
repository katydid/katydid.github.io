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

Given [a relatively large message](https://github.com/katydid/katydid/blob/master/relapse/tests/puddingmilkshake.proto).

We get the following benchmarks for the respective expressions listed below.

~~~
.FinanceJudo.SaladWorry.SpyCarpenter.BridgePepper:_ *= "a"		14905 ns/op	      0 allocs/op

.FinanceJudo.SaladWorry.SpyCarpenter:
	(.BridgePepper:_ *= "a" & .FountainTarget:_ *= "a") 		15572 ns/op	      0 allocs/op
~~~

That is tens of thousands of matches per second.


