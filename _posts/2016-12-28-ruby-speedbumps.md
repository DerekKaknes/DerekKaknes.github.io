---
layout: post
title:  "Ruby Speedbumps"
date:   2016-12-28
categories: jekyll update
---

Overall, I find that Ruby is a pretty enjoyable programming language to use - it's the
right amount of straightforwardness mixed with "fun" syntatic sugar.  As someone
who is more familiar with Python than Ruby (at the moment), I find that there
are less "super one-liners" that can take an hour to unpack mentally, which is
obviously a good thing, although something that I strangely find myself missing
(really, you miss [list
comprehension?][list-comp]).

One piece of syntatic sugar in Ruby that I still struggle with (a little) is the
optionality of paranthesis, particularly around function/method calls.  This is
not entirely new to me, as python2.7's print function operates in a similar way
(as do its unary operators), but I find that it can introduce some subtle
challenges, particularly when dealing with default argument values.  Take the
following example (which is trivial but illuminates the issue).  Assume that we
want to create a new method on the `Integer` class as follows:

{% highlight ruby %}
class Integer
  def add_value(val=1)
    total = self + val
    puts "{self} + {val} = {total}"
    total
  end
end
{% endhighlight %}

This new method simply takes an input of `val`, adds that value to the integer
on which it is called, and then prints and returns the sum.  The intrigue starts
because of the default value for `val`.  Take for example the following calls:

{% highlight ruby %}
2.add_value(2)
#=> prints "2 + 2 = 4", returns 4
3.add_value 3
#=> prints "3 + 3 = 6", returns 6
4.add_value
#=> prints "4 + 1 = 5", returns 5
4.add_value -1
#=> prints "4 + -1 = 3", returns 3
4.add_value - 1
#=> prints "4 + 1 = 5", returns 4
{% endhighlight %}

Wait, what happenned on that last one?  Well, a quick inspection can probably
identify the difference: Ruby interprets `-1` as `Integer(-1)`, whereas it
interprets `- 1` as the binary minus operator with a right argument of
`Integer(1)`.  That becomes important when combined with the default argument of
the preceeding `Integer#add_value` call - Ruby will pass the `Integer(-1)` as an
argument for `val` but it will not pass the `-` operator.  The obvious fix for
this is to never use `-1` when you actually mean `- 1`, but you can also
override the argument parsing with paranthesis, as follows:


{% highlight ruby %}
4.add_value(- 1)
#=> prints "4 + -1 = 3", returns 3
{% endhighlight %}

As I said, this is a pretty trivial case, but because the code will not raise
any errors, this type of bug can be pretty tricky to track down, particularly if
the method call is buried within a more complex set of functions.

[list-comp]: http://www.secnetix.de/olli/Python/list_comprehensions.hawk
