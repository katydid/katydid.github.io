---
layout: page
title: "Adding Functions"
category: dev
date: 2016-02-17
order: 2
---

Katydid does not allow the specification of new functions in Katydid itself.
Katydid is implemented in Go, so adding your own functions will require you to write some Go.

## Introduction

Let's look at the `contains` function:

{% highlight go %}

import (
	"strings"
	"github.com/katydid/katydid/funcs"
)

func Contains(s, sub String) Bool {
	return &contains{s, sub}
}

type contains struct {
	S      String
	Substr String
}

func (this *contains) Eval() (bool, error) {
	s, err := this.S.Eval()
	if err != nil {
		return false, err
	}
	subStr, err := this.Substr.Eval()
	if err != nil {
		return false, err
	}
	return strings.Contains(s, subStr), nil
}

func init() {
	funcs.Register("contains", new(contains))
}

{% endhighlight %}

As we can see there are four parts to this function, excluding the import part.

The first defines a publically accesible way to create the function.  This is only for users that are constructing their queries programmatically in Go.

The second defines the function parameters as a structure.
Each of these parameters have a type `funcs.String`, which is a very simple interface.

{% highlight go %}
package funcs

type String interface {
	Eval() (string, error)
}
{% endhighlight %}

This means that each parameter could be any structure or rather Katydid function that returns a string or even a string constant.

The second part is the `contains` struct's Eval method, the actual implementation of the function.
The method evaluates each of its parameters and then passes their values to the `strings.Contains` function which returns a `bool`.
We can now guess that `contains` implements the `funcs.Bool` interface.

{% highlight go %}
type Bool interface {
	Eval() (bool, error)
}
{% endhighlight %}

All function types are defined [here](https://github.com/katydid/katydid/blob/master/funcs/types.go).

Finally the `init` function registers the `contains` structure as a Katydid function.
The first parameter is the function name, since this can differ from the structure name.
This is especially useful when we want to do function overloading.

## Handling Errors

Some functions are inevitably going to have possible runtime errors.
Here we see an `elem` function which returns the element in the list at the specified index.

{% highlight go %}

type elemDoubles struct {
	List  Doubles
	Index Int
}

func (this *elemDoubles) Eval() (float64, error) {
	list, err := this.List.Eval()
	if err != nil {
		return 0, err
	}
	index64, err := this.Index.Eval()
	if err != nil {
		return 0, err
	}
	index := int(index64)
	if len(list) == 0 {
		return 0, NewRangeCheckErr(index, len(list))
	}
	if index < 0 {
		index = index % len(list)
	}
	if len(list) <= index {
		return 0, NewRangeCheckErr(index, len(list))
	}
	return list[index], nil
}

func init() {
	Register("elem", new(elemDoubles))
}

{% endhighlight %}

When we see that the function is trying to access an element outside of the list range, we simply return a plain go error.

### Handling Errors in Bool Functions

The not function does not return errors.
This means that when it receives an error it interprets it as false and returns true.

{% highlight go %}
func (this *not) Eval() (bool, error) {
	b, err := this.V1.Eval()
	if err != nil || !b {
		return true, nil
	}
	return !b, nil
}
{% endhighlight %}

This means that functions that are opposites of each other need to take this into account.
Here we can see the greater or equal function returning false given an error.

{% highlight go %}
func (this *intGE) Eval() (bool, error) {
	v1, err := this.V1.Eval()
	if err != nil {
		return false, nil
	}
	v2, err := this.V2.Eval()
	if err != nil {
		return false, nil
	}
	return v1 >= v2, nil
}
{% endhighlight %}

This means that the less than function needs to return true given an error.

{% highlight go %}
func (this *intLt) Eval() (bool, error) {
	v1, err := this.V1.Eval()
	if err != nil {
		return true, nil
	}
	v2, err := this.V2.Eval()
	if err != nil {
		return true, nil
	}
	return v1 < v2, nil
}
{% endhighlight %}

## Constants and Compile Time Evaluations

There are some functions for which you want to calculate some things only once, 
for example a regular expression matcher compiles the pattern only once.
Lets look at Katydid's builtin regex function.

{% highlight go %}
import (
	"regexp"
)

func Regex(e ConstString, s String) Bool {
	return &regex{Expr: e, S: s}
}

type regex struct {
	r    *regexp.Regexp
	Expr ConstString
	S    String
}

func (this *regex) Init() error {
	e, err := this.Expr.Eval()
	if err != nil {
		return err
	}
	r, err := regexp.Compile(e)
	if err != nil {
		return err
	}
	this.r = r
	return nil
}

func (this *regex) Eval() (bool, error) {
	s, err := this.S.Eval()
	if err != nil {
		return false, err
	}
	return this.r.MatchString(s), nil
}

func init() {
	Register("regex", new(regex))
}
{% endhighlight %}

There are a few new concepts here.
Firstly `r` is a field member of the struct, but it is not a parameter for the regex function.
Only struct fields with an `Eval` method are seen as function parameters.

Secondly `ConstString` is a type we have not encountered before.
`ConstString` is defined as exactly the same interface as String.
There is a corresponding constant type for each function type.
Katydid evaluates all functions, which do not depend on a variable, at compile time.
Variables will be discussed in the next section.
By specifying a parameter as a constant, you are explicitly stating that this parameter will be evaluated at compile time and if this is not possible it must result in a compile error.

Finally we get to the `Init` method, which compiles the regular expression at compile time, using the constant string and places the result in the field `r`.  
The `Eval` method then uses the compiled regular expression to match the bytes.

It is very important to remember to declare all the parameters you plan to use in the `Init` method as constant, otherwise you will get unexpected results.

## Variables

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

func Now() Int {
	return &now{}
}

type now struct{}

func (this *now) Eval() (int64, error) {
	return time.Now().UnixNano(), nil
}

func (this *now) IsVariable() {}

func init() {
	Register("now", new(now))
}
{% endhighlight %}

Obviously this function's value will be different almost every time that it is evaluated.
We added an  `IsVariable` method just to let Katydid know not to evaluate this function at compile time.


