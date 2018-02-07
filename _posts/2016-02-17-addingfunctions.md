---
layout: page
title: "Extendable"
category: relapse
date: 2016-02-17
order: 5
---

Relapse does not allow the specification of new functions in Relapse itself.
Relapse is implemented in Go, so adding your own functions will require you to write some Go.

## Introduction

Let's look at the implementation of the `contains` function:

{% highlight go %}
type contains struct {
	S           String
	Substr      String
	hash        uint64
	hasVariable bool
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

func (this *contains) Compare(that Comparable) int {
	if this.Hash() != that.Hash() {
		if this.Hash() < that.Hash() {
			return -1
		}
		return 1
	}
	if other, ok := that.(*contains); ok {
		if c := this.S.Compare(other.S); c != 0 {
			return c
		}
		if c := this.Substr.Compare(other.Substr); c != 0 {
			return c
		}
		return 0
	}
	return strings.Compare(this.String(), that.String())
}

func (this *contains) HasVariable() bool {
	return this.hasVariable
}

func (this *contains) String() string {
	return "contains(" + this.S.String() + "," + this.Substr.String() + ")"
}

func (this *contains) Hash() uint64 {
	return this.hash
}

func Contains(s, sub String) Bool {
	return TrimBool(&contains{
		S:           s,
		Substr:      sub,
		hash:        Hash("contains", s, sub),
		hasVariable: s.HasVariable() || sub.HasVariable(),
	})
}

func init() {
	Register("contains", Contains)
}

{% endhighlight %}

Our function is defined as a `struct`, which has to implement the following `Func` interface:

{% highlight go %}
type Func interface {
	Comparable
	HasVariable() bool
}

type Comparable interface {
	Compare(Comparable) int
	Hashable
	Stringer
}

type Hashable interface {
	Hash() uint64
}

type Stringer interface {
	String() string
}
{% endhighlight %}

Each function must also have an `Eval` method which returns a value of type: 
`string`, `[]byte`, `int64`, `uint64`, `bool` or `float64` and an `error`.

Our example is a function of type `Bool` so it implements an `Eval` method that returns a `bool` and an `error` and also the `Func` interface.
{% highlight go %}
type Bool interface {
	Func
	Eval() (bool, error)
}
{% endhighlight %}

Finally, the function is registered with its name and a constructor.

The constructor is responsible for:

  - placing the parameters in the `struct`, for later evaluation,
  - simplification of the function (optional),
  - trimming: calculating functions at compile time, if possible, and
  - pre-calculating a hash of the function and whether the function's parameters has a variable.

Each parameter is of type `Func`.  For our example, `contains`, both parameters are of type `String`, which is an interface which inherits the `Func` type.

{% highlight go %}
type String interface {
	Func
	Eval() (string, error)
}
{% endhighlight %}

Simplification is only done in rare cases like for the functions: `and`, `or` and `not`, 
so you won't need to worry about that.

Trimming tries to evaluate the function at compile time and replace it with a constant.
It can only trim pure functions, whose parameters also don't have variables.
This is why we calculate the `hasVariable` value, by checking whether any of the parameters have a variable.
If no parameter has a variable and our function is pure then it can be trimmed to a constant and should return `false` from its `HasVariable` method.
There is a `Trim<Type>` function for each type.

Pre-calculating a hash is done to speed up comparisons.
Comparisons are done as part of simplifications and minimizes the state space required by relapse.
These comparisons are quite expensive for large queries and so we use hashes to speed them up.
This is also why we want to calculate the hash only once.
The `Hash` helper function expects the function's name and its input parameters.

{% highlight go %}
func Hash(name string, hs ...Hashable) uint64
{% endhighlight %}

Now that we have defined the constructor, we can take a look at the other methods.
The `Hash` and `HasVariable` methods simply return the pre-calculated values.
These values are pre-calculated for efficiency reasons.

The `String` method should return the string representation of the function in its relapse syntax.
Parsing this string as an expression should result in the same function.

The `Compare` method is used for simplification of patterns and functions.
We first compare the Hash values, because if they differ then we don't need to do a deep comparison.
If the hash values are the same, then we need to look deeper.
The most likely case is that the function types are the same.
We can then compare the function's input parameters.
If they are all the same, then we should return `0` for equal.
If the types and hashes differ, then our last and most expensive resort is to create a string representation and compare these strings.
This is really expensive for large expressions and that is why we first try the hash and deep comparisons.

The `Eval` method evaluates each parameter and then using the resulting values does the actual function calculation and returns the value.

All function types are defined [here](https://github.com/katydid/katydid/blob/master/relapse/funcs/types.go).

Finally the `init` function registers the `contains` structure as a Relapse function.
The first parameter is the function name. 
This can differ from the structure name, which is especially useful when we want to do function overloading.

## Testing

Adding a user function is quite advanced behaviour and it is recommended that each function should be tested.
Here is an example test for the contains function:

{% highlight go %}

import (
	"strings"
	"testing"

	"github.com/katydid/katydid/parser/debug"
	"github.com/katydid/katydid/relapse/ast"
	c "github.com/katydid/katydid/relapse/combinator"
	"github.com/katydid/katydid/relapse/compose"
)

func TestContains(t *testing.T) {
	expr := ast.NewFunction("contains", c.StringVar(), c.StringConst("TheStreet"))
	b, err := compose.NewBool(expr)
	if err != nil {
		t.Fatal(err)
	}
	f, err := compose.NewBoolFunc(b)
	if err != nil {
		t.Fatal(err)
	}
	r, err := f.Eval(debug.NewStringValue("TheStreet"))
	if err != nil {
		t.Fatal(err)
	}
	if r != true {
		t.Fatalf("expected true")
	}
	r, err = f.Eval(debug.NewStringValue("ThatStreet"))
	if err != nil {
		t.Fatal(err)
	}
	if r != false {
		t.Fatalf("expected false")
	}
}

{% endhighlight %}

This will test that registration occurred correctly.

## Handling Errors

Some functions are inevitably going to have possible runtime errors.
Here we see an `elem` function which returns the element in the list at the specified index.

{% highlight go %}

type elemDoubles struct {
	List        Doubles
	Index       Int
	hash        uint64
	hasVariable bool
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

...


{% endhighlight %}

When we see that the function is trying to access an element outside of the list range, we simply return a plain go error.

### Handling Errors in Boolean Functions

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
Lets look at Relapse's built in regular expression function.

{% highlight go %}
import (
	"regexp"
)

func Regex(expr ConstString, input String) (Bool, error) {
	if expr.HasVariable() {
		return nil, fmt.Errorf("regex requires a constant expression as its first parameter, but it has a variable parameter")
	}
	e, err := expr.Eval()
	if err != nil {
		return nil, err
	}
	r, err := regexp.Compile(e)
	if err != nil {
		return nil, err
	}
	return TrimBool(&regex{
		r:           r,
		S:           input,
		expr:        e,
		hash:        Hash("regex", expr, input),
		hasVariable: input.HasVariable(),
	}), nil
}

type regex struct {
	r           *regexp.Regexp
	expr        string
	S           String
	hash        uint64
	hasVariable bool
}

func (this *regex) Eval() (bool, error) {
	s, err := this.S.Eval()
	if err != nil {
		return false, err
	}
	return this.r.MatchString(s), nil
}

...
{% endhighlight %}

We pre-calculate `r` in the constructor as a field member of the structure.
Our `Eval` method can then use the compiled regular expression to match the bytes.

We first check that `expr` does not have a variable, 
since we want to evaluate the expression at compile time.
In the constructor we specify that this parameter is a `ConstString`.
This is just a `String`, 
but it makes sure that the generated documentation mentions that this function requires the parameter to be calculated at compile time.

## Variables

Variables are values that possibly change with every execution, typically these are fields, but they can also include functions whose values change over time, database versions, etc.

If your function does not evaluate to the same value given the same parameters every time, you should declare it as variable.
Simply return `true` from the `HasVariable` method.

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

func (this *now) Hash() uint64 {
	return Hash("now")
}

func (this *now) Compare(that Comparable) int {
	if this.Hash() != that.Hash() {
		if this.Hash() < that.Hash() {
			return -1
		}
		return 1
	}
	if _, ok := that.(*now); ok {
		return 0
	}
	return strings.Compare(this.String(), that.String())
}

func (this *now) String() string {
	return "now()"
}

func (this *now) HasVariable() bool { return true }

func init() {
	Register("now", Now)
}
{% endhighlight %}

Obviously this function's value will be different almost every time that it is evaluated.

We make sure that `HasVariable` returns `true`, so that this function won't be trimmed and will return a different value for each evaluation.
