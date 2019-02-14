<h1 class="post-heading">Weird Ruby</h1>
<em>February 14, 2019 - [@kddeisz](https://twitter.com/kddeisz)</em>

Recently, I wrote a [plugin for `prettier`](https://github.com/prettier/plugin-ruby) for the Ruby programming language. Over the course of that process, I discovered a lot of eccentricities of Ruby (by needing to account for each node type of, as well as each variant in structure of, the AST). I found some fun things, and so in the spirit of Gary Bernhardt's ["wat" talk](https://www.destroyallsoftware.com/talks/wat), I'm going to share them with you here.

## case

`case` expressions in Ruby are much like switch statements in other languages. They provide a way to branch logic based on one common value. However, you can also have `case` expressions without a predicate at all, which makes them effectively `if..elsif` chains. As in:

```ruby
case
when foo == 1
  ...
when bar == 2
  ...
end
```

This is functionally equivalent to checking `if foo == 1` and `elsif bar == 2`.

## flip-flops

Flip-flops (the name coming from circuit design) have got to be one of the most esoteric operators in existence. [This article](https://blog.newrelic.com/engineering/weird-ruby-part-3-fun-flip-flop-phenom/) does a great job explaining it. Here's an example:

```ruby
lines.each do |line|
  next unless (line =~ start_pattern)..(line =~ end_pattern)

  puts line
end
```

Basically, this will loop through some list `lines` and will print out every line between the time that the first condition is met until the time that the second condition is met.

## for loops

Did you know Ruby still has `for` loops? I certainly didn't. They look like:

```ruby
for num in [1, 2, 3] do
  puts num
end
```

This also means that `in` is a reserved word, so you can't use it as an identifier.

## hooks

Ruby has two special blocks that are built into the language that allow you to hook into start and exit time of a script. I'd imagine they're particularly useful for scripting and less so for application-level logic. They look like:

```ruby
BEGIN {
  puts 'script has started'
}

END {
  puts 'script has ended'
}
```

Interestingly, these blocks don't support `do...end` syntax. I'm not really sure why.

## numbers

In Ruby there is a special syntax for numbers in binary, octal, and hex. All of the following statements evaluate to true:

```ruby
0b10 == 2 # this is a binary number
0o10 == 8 # this is an octal number
0x10 == 16 # this is a hex number
```

On top of that, you can even drop the `o` for octal numbers entirely and just use a `0` prefix, i.e. `010 == 8` is true. This one could pretty easily bite you if you don't watch out!

## procs

Procs and lambdas both have a special syntax for being called using square brackets. As in:

```ruby
add = ->(left, right) { left + right }
add[3, 4] # => 7
```

The number of arguments you pass into the square brackets aligns with the number that you pass into proc. So for procs or lambdas with an arity of zero, you can exclude arguments entirely, as in:

```ruby
greet = -> { puts 'Hello, world!' }
greet[]
```

## strings

Strings have all kinds of fun properties! Below are a couple of things by which I was surprised.

### `%x` literals

You probably know about backtick expressions (i.e., `` `ls` ``) which will spawn a process, execute it in the shell, and then return the stdio output back to the caller. (Be careful when doing this and please don't RCE yourself.)

You might not know about `%x` literals, which are actually the same thing (i.e., `%x[ls]`). Don't get confused with the other %-literals like `%w` and `%i` that create arrays!

### `%q` literals

You can use `%q` and `%Q` to create string literals (e.g., `%q[abc]`). These are effectively the same thing as using single quotes and double quotes, respectively, except that in the latter case you don't need to escape your double quotes. Convenient.

### Interpolation

Normal string interpolation in Ruby looks like `"a #{b} c"`, where `b` is some variable that responds to `to_s` (i.e., everything except `BasicObject`). But did you know that you can skip the braces if you're interpolating an instance, class, or global variable? In other word, the following code is entirely valid:

```
@instance = 'instance'
@@class = 'class'
$global = 'global'

"#@instance #@@class #$global" == 'instance class global' # => true
```

It even works inside regex expressions! If you replace the double quotes with `/` in the above expression you'll get a regex with interpolated variables.

### `?` literals

I've saved the best for last, as this was truly out of left field. Both of the following statements is valid in ruby:

```ruby
?a == 'a'
?\M-\C-a == "\x81"
```

As it turns out, the `?` character allows you to create strings of length 1, with a couple special extras thrown in. For further explanation, this is taken straight from the ruby docs:

```
  \cx or \C-x    control character, where x is an ASCII printable character
  \M-x           meta character, where x is an ASCII printable character
  \M-\C-x        meta control character, where x is an ASCII printable character
  \M-\cx         same as above
  \c\M-x         same as above
  \c? or \C-?    delete, ASCII 7Fh (DEL)
```

## void

In Ruby, you can use `()` to represent `nil`, as it's treated like an empty expression with no statements. This leads to the ability to do all kinds of weird things, like `!()` which evaluates to `true`.

## tl;dr

Ruby is so expressive that it provides you with many, many ways of doing things. This stands in stark contrast to many languages (for example in [The Zen of Python](https://www.python.org/dev/peps/pep-0020/) where it states "There should be one-- and preferably only one --obvious way to do it").

Of course this approach has tradeoffs. While it can provide you with the ability to ship code incredibly quickly, it can also slow down quick skimming if there are myriad approaches to doing things. This is one of the explicit goals of `prettier` and the Ruby plugin - we're trying to standardize so that you can get back to writing application code.

[← Back to home](/)
