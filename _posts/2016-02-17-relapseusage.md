---
layout: page
title: "Usage"
category: relapse
date: 2016-02-17
order: 1
---

This page will explain how to go from a Relapse grammar to validating a parsed tree.
This is done in two steps.
Each of these steps have several alternatives.

  1. Construct the Grammar
  2. Validation

This assumes you have already constructed a parser, if not see the page on [parser usage](../parser/addingparsers.html).

### Constructing a Relapse Grammar

First we need to parse our relapse string into a grammar.
This is perfect for when we are receiving a string from the user or a configuration file.

{% highlight go %}
import "github.com/katydid/katydid/relapse"

func main() {
	relapseString := "a:*"
	relapseGrammar, err := relapse.Parse(relapseString)
	if err != nil {
		panic(err)
	}
}
{% endhighlight %}

If you don't understand the relapseString's value it might be good idea to first take the [Relapse Tour](http://katydid.github.io/tour).

We could also have programmatically constructed our ast (abstract syntax tree).

{% highlight go %}
import "github.com/katydid/katydid/relapse/ast"

func main() {
	relapsePattern := ast.NewTreeNode(ast.NewStringName("a"), ast.NewZAny())
	relapseRefs := map[string]*ast.Pattern{"main": relapsePattern}
	relapseGrammar := ast.NewGrammar(relapseRefs)
}
{% endhighlight %}

This can become quite verbose, so we can rather use the combinator library, with which we can more concisely, but still programmatically, construct the grammar.

{% highlight go %}
import . "github.com/katydid/katydid/relapse/combinator"

func main() {
	relapsePattern := In("a", Any())
	g := G{"main": relapsePattern}
	relapseGrammar := g.Grammar()
}
{% endhighlight %}

We can also use this with the `funcs` library to get a type safe construction of a function.

{% highlight go %}
import . "github.com/katydid/katydid/relapse/combinator"
import . "github.com/katydid/katydid/funcs"

func main() {
	fun := NewStringEqual(NewStringVar(), NewStringConst("abc"))
	relapsePattern := In("a", Value(fun))
	g := G{"main": relapsePattern}
	relapseGrammar := g.Grammar()
}
{% endhighlight %}

### Validation

Now that we have a initialized parser and a relapse grammar we can validate the parsed tree against the schema.

{% highlight go %}
import "github.com/katydid/katydid/relapse"
import "github.com/katydid/katydid/relapse/ast"
import "fmt"

func validate(grammar *ast.Grammar, p parser.Interface) {
	match := relapse.Interpret(grammar, p)
	if !match {
		fmt.Printf("Expected match given %s\n", grammar.String())
	}
}
{% endhighlight %}

We could also compile the grammar for faster repeated execution.

{% highlight go %}
import "github.com/katydid/katydid/relapse"
import "github.com/katydid/katydid/relapse/ast"
import "fmt"

func validate(grammar *ast.Grammar, p parser.Interface) {
	compiled := relapse.Compile(grammar)
	match := relapse.Execute(compiled, p)
	if !match {
		fmt.Printf("Expected match given %s\n", grammar.String())
	}
}
{% endhighlight %}
