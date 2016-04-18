---
layout: page
title: "Usage"
category: parser
date: 2016-02-17
order: 1
---

This page will explain how to construct a parser and how to use that parser.

Currently katydid supports several parsers:

  - protobuf
  - json
  - reflected go structures
  - xml

### Constructing a Parser

A parser parses a specific serialization format or other parsable text or bytes.
We need to put our unparsed text, bytes or other structure into a parser so that it can be validated by relapse or transcoded to some other format.

{% highlight go %}
import "github.com/katydid/katydid/parser/json"

func main() {
	jsonString := `{"a": "abc"}`
	jsonParser := json.NewJsonParser()
	if err := jsonParser.Init([]byte(jsonString)); err != nil {
		panic(err)
	}
}
{% endhighlight %}

We can also construct a protocol buffer parser.

{% highlight go %}
import protoparser "github.com/katydid/katydid/parser/proto"
import "github.com/gogo/protobuf/proto"

func main() {
	protoParser := protoparser.NewProtoNameParser("mypackage", "mymessage", &Mymessage{}.Description())
	msg := &Mymessage{A: "abc"}
	data, err := proto.Marshal(msg)
	if err != nil {
		panic(err)
	}
	if err := protoParser.Init(data); err != nil {
		panic(err)
	}
}
{% endhighlight %}

We can even construct a parser for a reflected go structure.

{% highlight go %}
import reflectparser "github.com/katydid/katydid/parser/reflect"
import "reflect"

func main() {
	reflectParser := reflectparser.NewReflectParser()
	s := &MyStruct{A: "abc"}
	v := reflect.ValueOf(s)
	reflectParser.Init(v)
}
{% endhighlight %}

### Walking the Tree

If you want to use a parser for you own use case, here is a simple walk function.
{% highlight go %}
func walk(p parser.Interface) {
    for {
        if err := p.Next(); err != nil {
            if err == io.EOF {
                break
            } else {
                panic(err)
            }
        }
        value := print(p.(parser.Value))
        if !p.IsLeaf() {
            p.Down()
            walk(p)
            p.Up()
        }
    }
    return
}
{% endhighlight %}

We should always first call the Next method and check the error.
Next we can retrieve a value, which can be of type int, double, uint, string, bytes or a bool.
Lastly we can check whether the current position is a terminating leaf in the tree or if we can go down the tree and finally come back up.
Please see the godoc for the [parser.Interface](https://godoc.org/github.com/katydid/katydid/parser#Interface) and [parser.Value](https://godoc.org/github.com/katydid/katydid/parser#Value).
