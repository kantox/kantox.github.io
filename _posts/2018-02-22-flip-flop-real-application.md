---
layout: post
title: "Real applications of flip-flop in ruby"
description: "A couple of examples of flip-flop real applications"
category: hacking
tags: ruby
---

> ruby inherited the Perl philosophy of having more than one way to do the same
> thing. I inherited that philosophy from Larry Wall, who is my hero actually.
> I want to make ruby users free. I want to give them the freedom to choose.
> <small>Yukihiro Matsumoto</small>

Ruby surely inherited a flip-flop operator from Perl, amongst others. Why?
Because programming is sexy, ruby is sexy and—you get it—flip-flop is very sexy.

Uh. I probably should start with a brief explanation of what flip-flop is
in general. (There is another
[great writing](http://nithinbekal.com/posts/ruby-flip-flop/) on the subject by
Nithin Bekal.)

The term [came from the electronics](https://en.wikipedia.org/wiki/Flip-flop_(electronics))
where it roughly means two-state machine with memory. Basically, it’s meaning
can be expressed as a boolean variable that somehow depends on two conditions
_and_ the history. It’s initiated with `false`.
Once set to `true` because of the first condition, it remains
`true` until the second condition is evaluated to `true`. Now it turns back
to `false` until the first condition... I bet you got the point. Since it holds
the state and depends on it, flip-flop makes sense _only_ inside loops.

The simplest implementation of flip-flop in ruby (if it had no dedicated syntax,)
would be:

```ruby
def flip_flop(enum, cond1, cond2, fun)
  enum.inject(false) do |acc, value|
    acc ||= cond1.(value)
    fun.(value) if acc
    acc && !cond2.(value)
  end
end
```

and it might be used as:

```ruby
enum = [2,1,1,2,2]
cond1 = ->(i) { i.odd? }
cond2 = ->(i) { i.even? }
fun = ->(value) { print value }
flip_flop(enum, cond1, cond2, fun)
#⇒ 112
```

In the code above, it turned to have `truthy` state on the first _odd_ number
and turned back to `falsey` on the first even number. That simple.

### Flip-flip out of the box

The thing is ruby comes with a default syntax for flip-flop operator,
that looks a bit discouraging at first glance. It’s range operator with
left range boundary being a first condition, and the right one—being the second.

```ruby
cond1..cond2
```

Also, it is treated as flip-flop _only_ inside conditionals and ternary operator.
The example above might be rewritten using standard ruby syntax as:

```ruby
[2,1,1,2,2].each do |i|
  print i if i.odd?..i.even?
end
#⇒ 112
```

Cute, isn’t it?

### Real applications

Well, this is all sexy, but hey what’s the real purpose of this? This operator
is definitely not as common as most of others, but still.

Yesterday I needed to process a huge list of strings, finding matching pairs
where one string comes after another (not necessarily immediately) and counting
them. For those curious, it’s a sanity check for the FSM; basically I
stress-tested the FSM, printed out the states and after all I wanted to make
sure there were no state order violations (in this case the count should be zero
if everything is fine.)

For the sake of brevity I will use integers in the example below. So, let’s say
we have a huge list of integers and we want to count how many times `2` comes
after `1`.

```ruby
input = [3,1,2,2,1,1,1,3,2]
input.count { |i| i == 2 if (i==1)..(i==2) }
```

That’s basically it. Imagine, how clunky would look any other implementation,
when you have seen this one.

### Real applications ...more

- split ini-file into sections (yeah, I hear your screaming “regexp,” but still)
- split log file _on_ request IP / time change
- nearly everything that could be expressed as _blah-blah-blah-boom_.

I wish I could invent more of nifty examples, but as I said, this operator
is not of very common usage. Still, it’s a must-know of every Ruby developer
who pretends to have the Junior position outgrown.
