---
title: Dealing with unused variables
---

## How hard can it be?

Having unused variables is widely considered a code smell, and for good reasons:

- It can be a sign of [dead](https://refactoring.guru/smells/dead-code) [code](http://thecodelesscode.com/case/41).
- It can be a sign of mistakes made during implementation or refactoring. <sup>[citation needed]</sup>
- It can be unnecessary clutter, reducing readability.

That's why some languages will emit warnings about unused variables, like Elixir for example:

```elixir
iex(1)> fn (a) -> nil end
# warning: variable "a" is unused (if the variable is not meant to be used, prefix it with an underscore)
```

Even for languages that do not emit such warnings by default, there's usually a linter available with a [rule](https://phpmd.org/rules/unusedcode.html) [that performs](https://eslint.org/docs/rules/no-unused-vars) [this check](https://palantir.github.io/tslint/rules/no-unused-variable/).

So, let's say you come face to face with such a warning in a project you are working on. As a good developer that cares about the code quality of your project, you decide to apply the [Boy Scout Rule](https://deviq.com/boy-scout-rule/). Your linter tool of choice offers you a simple solution: remove the offending variable. You just need to follow it, right? Well, it's not so simple as it may seem at first. In order to evaluate possible solutions and the impact they may have, we need to make a distinction between a few different cases.


## Local variables

The simplest case to deal with is probably with local variables, but even this has a few gotchas.

Having unused local variables is usually a sign of dead code. The variable might have had some use in the past, but that's clearly no longer the case.

If the variable is only declared or initialized and never used again, the solution is often really simple, just remove the whole line:

```javascript
let a = "I'm no longer relevant :("; // Why this exists? Let's just remove it.
```

If the variable is not used after initialization but it's initialized with the result of a function call, the solution might not be so obvious:

```javascript
// Who knows what this function does?
// We better not remove it unless we are really sure.
let a = someFunction();
```

Of course we could keep the function call, but just not assign the result to a variable, right?

```javascript
someFunction(); // Cool! No more unused variables!
```

Not so fast! This will usually have no consequences, but it might:

In garbage collected languages, the value returned by `someFunction` might be garbage collected earlier than before, which might cause some unintended side-effects, like subtle changes in performance characteristics. But where it can really cause unexpected problems is in languages with [destructors](https://en.wikipedia.org/wiki/Destructor_(computer_programming)).

This PHP example:

```php
<?php

class ClassWithDestructor {
  function __destruct() {
    print("destroying " . __CLASS__ . "\n");
  }
}

function giveMeAnInstance() {
  return new ClassWithDestructor();
}

function assignResult() {
  print("calling assignResult:\n");
  $result = giveMeAnInstance();

  sleep(1);

  print("done!\n");
}

function ignoreResult() {
  print("calling ignoreResult:\n");
  giveMeAnInstance();

  sleep(1);

  print("done!\n");
}

assignResult();
echo("\n");
ignoreResult();
```

Will (usually) result in the following output:

```
calling assignResult:
done!
destroying ClassWithDestructor

calling ignoreResult:
destroying ClassWithDestructor
done!
```

As you can see, the order of execution will be completely different between `assignResult` and `ignoreResult` with the only difference between the two functions being that in one we assign the result of `giveMeAnInstance` to an **unused** variable and in the other we don't.

You should ask yourself a few questions before removing an unused local variable:

- Is the function [pure](https://en.wikipedia.org/wiki/Pure_function)? If it is you can safely remove the variable **and** the function call.
- Are the side effects of the function covered by a test case? If they are not, you should probably do it before removing the function.
- Does the order of execution of side-effects (like garbage collection, or code triggered by a destructor) matter? If they do you should probably keep the variable.


## Public / global variables

It probably sounds obvious, but a lot more care needs to be taken in this case. It's not as easy for automated tools to check for unused public variables and it might generate false positives. Especially if the language has some [crazy feature](https://www.php.net/manual/en/language.variables.variable.php) that basically works like a simpler `eval`. Having good test coverage can help you, but things can still go wrong in a variety of ways. I'm not going to say much more here. How safe you are will depend on your codebase.


## Function arguments

Unused function arguments can also be the result of changes in the codebase, but it's a completely different case. It's not exactly dead code since there's no code **not** being executed. The function is probably still receiving the argument, it's just not doing anything with it. Why is it bad then? Well, it *might* need to be used, someone just forgot to. This means that it might also be a false positive, you could simply not need to use it. Well, let's look at a few specific cases:

### Private functions

Private functions are usually easy to deal with. You can probably change all the callers to no longer pass the unused argument and safely remove it from the function. Unless your language has a way of [calling private functions from outside](https://rubyquicktips.com/post/2681282667/how-to-call-private-methods-on-objects) or has a [weird concept of private](https://www.crockford.com/javascript/private.html), in that case, good luck. All callers are already covered by tests anyway, right? No? Why are you even considering removing the argument? Implement the tests first!

### Public functions

Changing the signature of public functions is a lot more dangerous than changing private functions. A lot of code in your codebase might depend on the function and will probably still be sending the unused argument. [Some](https://stackoverflow.com/questions/12694031/what-happens-if-i-call-a-js-method-with-more-parameters-than-it-is-defined-to-ac) [languages](https://wiki.php.net/rfc/strict_argcount) will hapilly accept calls with extra undeclared arguments, so you might be tempted to just remove the argument from the function declaration and let the callers pass the extra unused argument. Abusing a quirk of the language like this has dangerous consequences. I have seen cases like this happen a few times in PHP codebases:

```php
<?php

function myReallySpecialSum($a, $b, $reallyImportantExtraArgument) {
  doSomethingImportantWithMyImportantArgument($reallyImportantExtraArgument);
  return $a + $b;
}

// Meanwhile, in a different part of the code:

// What does `true` even mean here?
myReallySpecialSum(1, 1, true);
// Oh well, that's not the point of the example, let's keep going.
```

Some time later:

```php
<?php

// Turns out we don't really need `$reallyImportantExtraArgument`,
// let's just remove it.
function myReallySpecialSum($a, $b) {
  return $a + $b;
}

// Meanwhile, in several places of the codebase:

myReallySpecialSum(1, 1, true);

// ...

myReallySpecialSum(1, 2, false);

// ...

myReallySpecialSum(4, 4, $aBadlyNamedVariable);

// The language is not complaining about the extra argument
// and I don't want to change every single call, let's keep them there.
```

Even more time later, in a new part of the codebase:

```php
// Hmm, `myReallySpecialSum` only takes 2 arguments, alright!
myReallySpecialSum(2, 2);

```

Later still:

```php
// Hmm, I really need a new argument here,
// but I don't want to mess with people already calling this function.
// I know, let's make it optional!
function myReallySpecialSum($a, $b, $newOptionalArg = null) {
  if ($newOptionalArg) {
    doSomeCoolNewThing();
  }

  return $a + $b;
}
```

And now you have a huge mess. You have calls with just 2 arguments, calls with 3 arguments expecting it to be `$reallyImportantExtraArgument` and calls with 3 arguments expecting it to be `$newOptionalArg`. Your code now has a very hard to track down bug. Even with decent test coverage you might still have issues. Calls that expect the extra argument to be either `$reallyImportantExtraArgument` or `$newOptionalArg` could use the same value by coincidence. Yes, this could be somewhat prevented with type checking, but only if `$reallyImportantExtraArgument` and `$newOptionalArg` had different types.



### Interfaces and subclasses

Having unused arguments in implementations of an interface can be evidence that the interface is just too broad, a [god interface](https://en.wikipedia.org/wiki/God_object) if you will. In that case you might want to refactor it into smaller interfaces. But it might also be that some implementations simply have no use for an argument. If the argument makes sense for most of the implementations but are not used by a few exceptions, it might not be worth it changing the whole codebase just to keep some exceptional implementation from having unused arguments. In that case, it's a good practice to make it explicit that this implementation is in some way exceptional. A good example of that is when implementing [test doubles](https://blog.pragmatists.com/test-doubles-fakes-mocks-and-stubs-1a7491dfa3da), specially stubs:

```php
// Pardon the lack of type hints, it's not relevant for the example.
interface HttpClient {
  public function get($url);
}

// A stubbed version created in some test:
class HttpClientForTest implements HttpClient {
  public function get($url) {
    return [200, "OK!"];
  }
}
```

`HttpClientForTest` is a stubbed implementation that completely ignores the `$url` argument. It's explicitly named with the `ForTest` suffix, so it would be pretty obvious that something is wrong if this class was found being used outside of tests. "OK, but what about the unused `$url` argument", you might say, "my linter is still screaming about it!". For languages that have the "feature" of allowing calls with a different number of arguments, the fix suggested by the linter might be to remove the argument. I strongly suggest against it for a few reasons:

- It violates the [principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment). A developer reading the test code might be baffled seeing the `get` function. "What? How come it has no arguments?"
- Having a variable there, even when not used, serves as documentation for future developers reading the code. The value is still being passed by the caller, we are just deciding not to use it.
- There are [documented cases](https://github.com/microsoft/TypeScript/issues/32196) of the linter / IDE integration removing an argument from places where it will definitely cause bugs, like in the middle of a function declaration.

It's hard to come up with real examples that don't involve tests because, as mentioned, we are dealing with exceptional cases. What can be considered either an acceptable exception or a case of bad design will depend greatly on specific details of your codebase. Still, making it explicit that we are ignoring an argument should make it clear for developers reading the code that it's intentional and not a mistake. They might still question if there's a better solution, but at least they know it's by design.

How do I suggest we make it explicit then? Well, let's look at one more case before that.


### Callbacks

There are [some cases](https://github.com/microsoft/TypeScript/issues/17868) where the community considers it acceptable to omit unused arguments on callbacks, specially for callbacks of functions of the standard library. A commonly mentioned case is [Array.prototype.map()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) in Javascript, which has a callback with 2 arguments usually not used at all. It's probably ok to omit the unused arguments in this case since it's a very well known function signature, but think about it this way: not every developer might know about the optional callback arguments. Ask yourself a few questions:

- Is it easy for an inexperienced developer to look up the function definition or documentation and learn about the optional arguments?
- Is the omitted argument a common enough use case, like needing the `index` argument on a `map` callback, that it would make it easier for developers to know the argument is there, just not being used?

Either way, I'd rather err on the side of explicitness than to obfuscate code by leaving important details implicit.


## A simple solution for most cases

After taking everything under consideration you might decide that removing the variable is a completely sane solution. But for some cases you decide it's either too dangerous or it would help document the code keeping it there. In that case, how do I make the linter shut up? Should I completely disable this check? You could, but you would be potentially allowing your codebase to accumulate the bad kind of unused variables. You could disable the check for a single file, but then it would not be clear where that unused variable is on the file. You could disable the check for a single line, but it may still not be clear which variable is unused in functions with more than one argument.

The main issue here is this: a linter is a tool that helps the developer writing the code, but code is [read much more often](https://www.goodreads.com/quotes/835238-indeed-the-ratio-of-time-spent-reading-versus-writing-is) than it's written. If you really want to help the readers of your code know that a variable is unused intentionally, you need to be explicit about it. There's a somewhat controversial article by Joel Spolsky called [Making Wrong Code Look Wrong](https://www.joelonsoftware.com/2005/05/11/making-wrong-code-look-wrong/) where he suggests using a kind of Hungarian Notation to help developers identify "wrong code". I'm not sure I completely agree with everything he says, but he has a good point: adding visual cues to variable names can help a developer quickly identify potentially dangerous pieces of code.

So, what's my suggested solution? Most linter tools can be configured to skip the unused variable check for variable names that follow some pattern. I particularly like the way that Elixir handles unused variables (and Elixir in general, but that's another story): by adding a `_` at the beginning of the name, the following will happen:

- The compiler will shut up about having an unused variable.
- The developer writing the code will be saying: "Yes, this variable is unused, but I know what I'm doing!".
- The developer reading the code will be able to, at a glance, say: "Hmm, so this variable is unused. What are the implications?".

In [some](https://stackoverflow.com/questions/4484424/underscore-prefix-for-property-and-method-names-in-javascript) [languages](https://stackoverflow.com/questions/1641219/does-python-have-private-variables-in-classes) that don't have the concept of `private` there's sometimes a convention of using `_` at the beginning of a name to denote a "private" member. But that's only a convention and does not bring any of the benefits of actual encapsulation, since there's nothing stopping it from being used publicly except for developer discipline. For Javascript at least there's an easy way to implement proper encapsulation by using closures, so I advise against using this convention if possible. Even Douglas Crockford, the author of *JavaScript: The Good Parts*, [recommends the following](https://www.crockford.com/code.html): "Do not use _ *underbar* as the first or last character of a name. It is sometimes intended to indicate privacy, but it does not actually provide privacy. If privacy is important, use closure. Avoid conventions that demonstrate a lack of competence.". I'm not sure he would agree with my suggestion of using `_` for unused variables, but I don't think he would suggest doing so to document intentionally unused variables demonstrates a "lack of competence".


## Conclusion

We are finally reaching the end of the article and you might be thinking: "Really, did you just write this huge article about such a trivial issue to in the end tell me to just add an `_` at the beginning of variables?". Well, when you put it like that... But no, I don't think it's really that trivial. I only superficially explored the impact a single linter rule can have on a codebase. What I really want you to get out of this is that the tools we choose can have subtle and unintended consequences on the quality of our code. Simply following a convention because it's there, or because the community or an influential developer said so can be a bad idea if we don't stop to think about the motivations and consequences. We risk falling into a [cargo cult](https://en.wikipedia.org/wiki/Cargo_cult_programming) mentality. Do follow good practices and conventions, but be aware that conventions change and that even things widely considered good practices can have weak points. Maybe you can be the one to identify a potential weakness in a convention your project uses and change it for the better. And who knows, if you write a nitpicky article about it and be really loud about it, the community might even start adopting it ;)
