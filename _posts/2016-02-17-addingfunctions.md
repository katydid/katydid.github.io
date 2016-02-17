---
layout: page
title: "Adding Functions"
category: doc
date: 2016-02-17
order: 5
---

*Out of Date*

Katydid does not allow the specification of new functions in Katydid itself.
Katydid is implemented in Go, so adding your own functions will require you to write some Go.

Introduction
------------

Let's look at the `contains` function:

{% highlight go %}

import (
	"strings"
	"github.com/katydid/katydid/funcs"
)

type contains struct {
	S      funcs.String
	Substr funcs.String
}

func (this *contains) Eval() bool {
	return strings.Contains(this.S.Eval(), this.Substr.Eval())
}

func init() {
	funcs.Register("contains", new(contains))
}

{% endhighlight %}

As we can see there are three parts to this function, excluding the import part.

The first defines the function parameters as a structure.
Each of these parameters have a type `funcs.String`, which is a very simple interface.

{% highlight go %}
package funcs

type String interface {
	Eval() string
}
{% endhighlight %}

This means that each parameter could be any structure or rather Katydid function that returns a string or even a string constant.

The second part is the `contains` struct's Eval method, the actual implementation of the function.
The method evaluates each of its parameters and then passes their values to the `strings.Contains` function which returns a `bool`.
We can now guess that `contains` implements the `funcs.Bool` interface.

{% highlight go %}
type Bool interface {
	Eval() bool
}
{% endhighlight %}

All function types are defined [here](https://github.com/katydid/katydid/blob/master/funcs/types.go).

Finally the `init` function registers the `contains` structure as a Katydid function.
The first parameter is the function name, since this can differ from the structure name.
This is especially useful when we want to implement the same function for multiple input parameters.

Handling Errors
---------------

Some functions are inevitably going to have possible runtime errors.
When executing a Katydid recognizer we expect it to return true or false and a possible error.
Having your error return from the recognizer is as easy as throwing it.
Here we see an `elem` function which returns the element in the list at the specified index.

{% highlight go %}

type elemFloat64s struct {
	List  Float64s
	Index Int64
	Thrower
}

func (this *elemFloat64s) Eval() float64 {
	list := this.List.Eval()
	index := int(this.Index.Eval())
	if len(list) == 0 {
		this.Throw(errors.New("list is empty"))
		return 0
	}
	//allows negative indices
	if index < 0 {
		index = index % len(list)
	}
	if len(list) <= index {
		this.Throw(NewRangeCheckErr(index, len(list)))
		return 0
	}
	return list[index]
}

func init() {
	Register("elem", new(elemFloat64s))
}

{% endhighlight %}

`Thrower` is an instance of a struct rather than an interface. 
Embedding, as in this example, initialises the Thrower struct to its default values and adds its exported methods to the `elemFloat64s` struct.
These exported methods allow the `elemFloat64s` struct to satisfy the `SetCatcher` interface, which Katydid will be looking for when compiling your error throwing function.  
This means that if you do not embed `Thrower` your function will not be able to throw an error.

Our first error happens when we try to take an element from an empty list.

{% highlight go %}
if len(list) == 0 {
	this.Throw(errors.New("list is empty"))
	return 0
}
{% endhighlight %}

This creates a new error and throws it to a catcher which is implemented as part of Katydid.

The `Thrower` has a few helper functions to make the code more concise.

{% highlight go %}
if len(list) == 0 {
	return this.ThrowFloat64(errors.New("list is empty"))
}
{% endhighlight %}

`ThrowFloat64` throws an error just like before but also returns a zero float64 value.

Our second error throws a range check error, which can also be written more concisely.
{% highlight go %}
if len(list) <= index {
	return this.ThrowFloat64(NewRangeCheckErr(index, len(list)))
}
{% endhighlight %}

`NewRangeCheckErr` is just a function which returns an error.

{% highlight go %}
type ErrRangeCheck struct {
	Index int
	Len int
}

func (this ErrRangeCheck) Error() string {
	return fmt.Sprintf("range check error: trying to access index %d in list of length %d", this.Index, this.Len)
}

func NewRangeCheckErr(index, l int) ErrRangeCheck {
	return ErrRangeCheck{index, l}
}
{% endhighlight %}

This is just plain Go, anything that satisfies the error interface qualifies as an error.

{% highlight go %}
type error interface {
	Error() string
}
{% endhighlight %}

Constants and Compile Time Evaluations
--------------------------------------

There are some functions for which you want to calculate some things only once, 
for example a regular expression matcher compiles the pattern only once.
Lets look at Katydid's builtin regex function.

{% highlight go %}
import (
	"regexp"
)

type regex struct {
	compiledRegex *regexp.Regexp
	Expr ConstString
	B    Bytes
}

func (this *regex) Init() error {
	r, err := regexp.Compile(this.Expr.Eval())
	if err != nil {
		return err
	}
	this.compiledRegex = r
	return nil
}

func (this *regex) Eval() bool {
	return this.compiledRegex.Match(this.B.Eval())
}

func init() {
	Register("regex", new(regex))
}
{% endhighlight %}

There are a few new concepts here.
Firstly `compiledRegex` is a field member of the struct, but it is not a parameter for the regex function.
Only struct fields with an `Eval` method are seen as function parameters.

Secondly `ConstString` is a type we have not encountered before.
`ConstString` is defined as exactly the same interface as String.
There is a corresponding constant type for each function type.
Katydid evaluates all functions, which do not depend on a variable, at compile time.
Variables will be discussed in the next section.
By specifying a parameter as a constant, you are explicitly stating that this parameter will be evaluated at compile time and if this is not possible it must result in a compile error.

Finally we get to the `Init` method, which compiles the regular expression at compile time, using the constant string and places the result in `compiledRegex`.  
The `Eval` method then uses the compiled regular expression to match the bytes.

It is very important to remember to declare all the parameters you plan to use in the `Init` method as constant, otherwise you will get unexpected results.

Variables
---------

Variables are values that possibly change with every execution, typically these are fields, but they can also include functions whose values change over time, database versions, etc.

If your function does not evaluate to the same value given the same parameters every time, you should declare it as variable.
Simply make sure your function satisfies the variable interface.

{% highlight go %}
type Variable interface {
	IsVariable()
}
{% endhighlight %}

Lets look at the `now` function:

{% highlight go %}
import "time"

type now struct{}

func (this *now) Eval() int64 {
	return time.Now().UnixNano()
}

func (this *now) IsVariable() {}

func init() {
	Register("now", new(now))
}
{% endhighlight %}

Obviously this function's value will be different almost every time that it is evaluated.
We added a  `IsVariable` method just to let Katydid know not to evaluate this function at compile time.

Injecting Values
----------------

Sometimes you have a function that is dependant on a value that changes often, 
but you don't want to create a global variable which this function can access.
You would prefer to inject this value into your function.

Using the [Implements](http://godoc.org/github.com/katydid/katydid/asm/inject) function in the `inject` package you can retrieve all Katydid functions that satisfies a specific interface as a list.
You can then range over this list calling a type of `Set` method to inject your value.

This is quite an advanced function.
Please see the [inject test](https://github.com/katydid/katydid/blob/master/asm/test/inject_test.go) for an example.

