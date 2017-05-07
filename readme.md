# ember-workshop

This is a workshop designed for intermediate developers who have worked on at least one large Ember app. The goal is to teach functional programming basics and to be able to use these ideas in your app.

This document contains the notes that I am reading from / referring to during the workshop. Ideally if you use this as a basis for your own workshop, you will include interactive live coding sessions as well.

* [Part 1 - FP Basics](#part-1---fp-basics)
  + [FP in JavaScript](#fp-in-javascript)
    - [What is FP?](#what-is-fp-)
    - [Examples](#examples)
    - [`Array.map`](#-arraymap-)
    - [`Array.reduce`](#-arrayreduce-)
    - [The `pipe` function](#the--pipe--function)
    - [The `compose` function](#the--compose--function)
    - [Implementing the `pipe` function](#implementing-the--pipe--function)
    - [Currying](#currying)
  + [CPs](#computed-properties)
    - [Computed property macros](#computed-property-macros)
  + [Helpers](#helpers)
* [Part 2 - Testing](#part-2---testing)
  + [Unit tests](#unit-tests)
    - [Anatomy of a unit test](#anatomy-of-a-unit-test)
    - [Testing philosophy](#testing-philosophy)
    - [Basic TDD](#basic-tdd)
    - [Data-driven testing](#data-driven-testing)
    - [Testing computed property macros (or anything that requires an `Ember.Object`)](#testing-computed-property-macros--or-anything-that-requires-an--emberobject--)
  + [Integration tests](#integration-tests)
    - [Why do we set values and actions on the test context?](#why-do-we-set-values-and-actions-on-the-test-context-)
    - [`this.$()`](#-this----)
    - [Dependency injection in components](#dependency-injection-in-components)
  + [Acceptance tests](#acceptance-tests)
    - [Mocking API responses](#mocking-api-responses)
* [Part 3 - ember-concurrency](#part-3---ember-concurrency)
  - [Task syntax](#task-syntax)
  - [Task concurrency and cancelation](#task-concurrency-and-cancelation)
  - [Examples](#examples-1)

## Part 1 - FP Basics

### FP in JavaScript

Warning: MATH!

#### What is FP?
  + Programming paradigm based on mathematical functions
    * Referential transparency - 1 input maps to 1 output
    * `f(1) = A`
    * `f(2) = B`
    * `f(3) = C`
  + Immutability
    * Minimizing side-effects and mutating state
  + First class functions
    * Functions are first class
    * Can be passed as arguments
  + Recursion
    * Usually plays well with a FP paradigm
  + JavaScript is not a functional language
    * But you can do some functional things with it
  + Contrast to something like Haskell
    * Strongly typed
    * Compiled language

#### Examples

Basic example of contrasting imperative to functional programming:

```js
// imperative
const data = [1, 2, 3];

for (let i = 0; i < data.length; i++) {
  data[i] = data[i] * 3;
}

console.log(data); // [3, 6, 9]
```

#### `Array.map`

With a FP minded approach, we prefer using `map` when we need to run a function on each item in an array.

```js
// functional
const data = [1, 2, 3];
const result = data.map((n) => n * 3); // [3, 6, 9]

console.log(data); // [1, 2, 3]
console.log(result); // [3, 6, 9]
```

This approach leaves the original array intact, and is simpler to read and reason about. As an additional benefit, since it returns a new array, you can chain additional array methods:

```js
const isEven = (n) => n % 2 === 0;
const result = [1, 2, 3]
  .map((n) => n * 3)
  .filter(isEven);

console.log(result); // [6]
```

You'll notice that I made an anonymous function and bound it to the variable `isEven`. Since functions are first class in JavaScript, I can simply pass the function into `filter`, instead of defining the anonymous function inline.

#### `Array.reduce`

The interesting thing about all these array methods like `map`, `filter`, `reject`, `find` and so on is that they can actually all be derived from a single method called `reduce`:

```js
const map = (arr, f) => {
  return arr.reduce((acc, curr) => [...acc, f(curr)], []);
};
```

If this looks weird, fret not. Let's first look at what `reduce` does. According to MDN:

> The reduce() method applies a function against an accumulator and each value of the array (from left-to-right) to reduce it to a single value.

Let's look at how we implemented `map` using `reduce`. The `reduce` function takes this argument signature:

```js
array.reduce(callback, initialValue);
```

An accumulator is simply a value that is carried forward each iteration of the items in the array. Since `map` should return a new array, we use an empty array literal as the initial value. Now, let's look at the callback function we pass into `reduce`.

The callback function takes a number of arguments:

```js
const exampleCallback = (accumulator, currentValue, currentIndex, array) => {
  // accumulator is the initial value carried forward
};
```

Whatever is returned in the callback becomes the new `accumulator` in the next iteration. So the first time the function is run, `acc` is just an empty array.

The `[...acc, f(curr)]` syntax is ES2015. The `...` is the spread operator, which basically means that we want to get the list of values in the `acc` array, then "unwrap" them. So if `acc` is currently `[2, 4, 6]`, `[...acc, 8]` is the equivalent of writing `[2, 4, 6, 8]` by hand.

What this means is that `(acc, curr) => [...acc, f(curr)]` we return a new array with the accumulator's current values, and add a new value to the end of the array which is the result of running the callback function on the current item in the iteration.

`reduce` is a very powerful method, and you can basically write almost any array method with it. You can write a custom `find` function, and so forth.

You can even construct powerful functions with it. For example, let's look at the `pipe` function.

#### The `pipe` function

Let's say we have a number of math functions, like so:

```js
const square = (x) => x * x;
const half = (x) => x / 2;
const triple = (x) => x * 3;
```

And say we wanted to apply it to some value like so:

```js
const result1 = square(10);
const result2 = half(result1);
const result3 = triple(result2);
```

Pretty verbose! I guess you could write it all in one-line:

```js
const result = triple(half(square(10)));
```

But who wants to read something like that, right?

Well, since you can pass functions around in JavaScript, let's see what we can do better. In FP, functions can be composed. What that means is that we can create new functions from multiple other functions, also known as [function composition](https://en.wikipedia.org/wiki/Function_composition). Let's see the most basic example of this.

#### The `compose` function

Let's call this function `compose`.

```js
const compose = (f, g) => (x) => f(g(x));
```

This might look very terse, and it is. Let's explode it into normal ES5 syntax:

```js
var compose = function compose(f, g) {
  return function (x) {
    return f(g(x));
  };
};
```

This version might be easier to understand for those of you still new to ES2015 syntax. The `compose` function takes 2 arguments `f` and `g`. It expects that they are both functions themselves, and then returns a new function which expects one argument `x`. 

That function then returns the value of `g(x)` passed into the `f` function. The way this works is that we have created a closure - even though the new function only takes 1 argument, we were able to "store" the previous function arguments and then remember it later when we do invoke it. 

Let's see how we can use this:

```js
const compose = (f, g) => (x) => f(g(x));
const square = (x) => x * x;
const half = (x) => x / 2;

const squareAndHalf = compose(half, square);
console.log(squareAndHalf(10)); // 50
```

You might notice that I called the composed function `squareAndHalf`, but the arguments are passed in reverse - that is because `compose` composes functions from right to left.

With that, I hope it further illustrates how functions are truly first class in JavaScript. Let's take the `compose` function example a little bit further and introduce the `pipe` function, which does function composition as well but passes from left to right.

#### Implementing the `pipe` function

To write this function, let's revisit the array `reduce` method:

```js
const pipe = (fn, ...fns) => (...args) => fns.reduce((acc, f) => f(acc), fn(...args));
```
[(source)](https://medium.com/@dtipson/functional-lenses-d1aba9e52254#.kff3l5d6j)

Now, you can write create a new function by using `pipe`, which takes a bunch of functions as arguments and returns a new function:

```js
const customFunction = pipe(square, half, triple);
console.log(customFunction(10)); // 150
```

How does this work? Let's see what the transpiled version looks like:

```js
var pipe = function pipe(fn) {
  for (var _len = arguments.length, fns = Array(_len > 1 ? _len - 1 : 0), _key = 1; _key < _len; _key++) {
    fns[_key - 1] = arguments[_key];
  }

  return function () {
    return fns.reduce(function (acc, f) {
      return f(acc);
    }, fn.apply(undefined, arguments));
  };
};
```

The first `for` loop is the transpiled output for the `rest` operator, which is the first part `const pipe = (fn, ...fns)`. This means that the first argument is bound as the parameter `fn`, and all the rest of the arguments (any number aka variadic) is bound to an array called `fns`. 

Now we need to return a function. This function should be variadic and take any number of arguments as well, so it can support anything. This is the next bit: `=> (...args)`.

Finally, in our new function, we `reduce` over the array of functions starting from `n + 1`. And we use the result of the first function applied to those arguments as the initial value, so that we can pass that value along to the next functions. The body of the actual function becomes simple, we just return the value of the current function in the iteration applied to the accumulator. 

So the first iteration of `pipe` looks something like this (pseudo-code, for explanatory purposes only):

```js
return [half, triple].reduce((acc, f) => f(acc), square(10));
```

The square of 10 is `100`: 

```js
return [half, triple].reduce((acc, f) => f(acc), 100);
```

Next iteration:

```js
return [triple].reduce((acc, f) => f(acc), half(100));
```

Half of 100 is `50`:

```js
return [].reduce((acc, f) => triple(50));
```

And finally the triple of 50 is `150`. Since we have run out of values to reduce, we just return the final accumulator.

Pretty cool right?! But how would you use this in your application? Well, in Ember, closure actions are just functions:

```hbs
{{my-component someAction=(action "someAction") otherAction=(aciton "otherAction")}}
```

If you use `ember-composable-helpers`, you can use the `pipe` helper to compose actions:

```hbs
<button {{action (pipe addToCart purchase redirectToThankYouPage) item}}>
  1-Click Buy
</button>
```

This means that your actions can be much simpler instead of having 1 big action that mixes business logic with presentational logic.

#### Currying

[Currying](https://en.wikipedia.org/wiki/Currying) is a concept in FP where you can make a function accept its arguments one at a time. For example, if function `f` is ternary (arity of 3), and we make a new curried function `g`, the following are equivalent:

```js
g(1)(2)(3);
g(1)(2, 3);
g(1, 2)(3);
g(1, 2, 3);
```

Here is a simplified example of a curry function (it does not handle all cases properly):

```js
const curry = (f, ...args) => (f.length <= args.length) ? 
  f(...args) : 
  (...more) => curry(f, ...args, ...more);
```
[(source)](https://medium.com/@dtipson/functional-lenses-d1aba9e52254#.kff3l5d6j)

And a simple `add` function:

```js
const add = (x, y, z) => x + y + z;
```

When `add` is curried, it now has curried capabilities:

```js
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1)(2, 3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
console.log(curriedAdd(1, 2, 3)); // 6
```

`curry` is variadic and expects a function as the first argument, and an optional list of arguments to be provided to that function. This function is recursive.

When writing a recursive function, the first thing you want to think of is the base case - the base case refers to the point where the recursion is "complete" and the function returns an optional value. Without a base case, your recursive function becomes an infinite loop.

Let's look at the base case first:

The function's length is checked (`Function.length` returns the number of arguments expected by the function), and if it is less than or equal to the length of the remaining arguments, it means that all arguments were passed in and we can simply run the function with the given arguments, i.e. `add(1, 2, 3)`.

If not, we return a new variadic function. This function then calls `curry` again recursively, passing in the same function `f`, but also passing along the list of `args` as well as `more` into curry. This is a more readable version:

```js
const curry = (f, ...args) => {
  // base case, all arguments provided
  if (f.length <= args.length) {
    return f(...args);
  }
  
  // recursive
  return (...more) => {
    return curry(f, ...args, ...more);
  };
};
```

Currying is useful because you get easier reuse of more abstract functions, since you get to specialize. Let's look at that `add` function again. With currying, we can create very specialized functions:

```js
const curry = (f, ...args) => (f.length <= args.length) ? 
  f(...args) : 
  (...more) => curry(f, ...args, ...more);
const add = (x, y, z) => x + y + z;

const addOne = curry(add, 1);
const addFiveAndThree = curry(add, 5, 3);

console.log(addOne(4, 5)); // 10
console.log(addFiveAndThree(2)); // 10
```

This is similar to the function composition we saw earlier, but instead of composing functions we are in a sense composing arguments and creating preset functions that do specific things from a more abstract function.

[GIST](https://ember-twiddle.com/ad7133a47b104f77cc484e647b98b861?openFiles=templates.components.foo-bar.hbs%2C)

In Ember we can do something similar with closure actions. The same `add` function, now as an action:

```js
import Ember from 'ember';

const { Controller } = Ember;

export default Controller.extend({
  actions: {
    add(x, y, z) {
      return x + y + z;
    }
  }
});
```

```hbs
{{foo-bar addOne=(action "add" 1) addFiveAndThree=(action "add" 5 3)}}
```

We have curried the `add` action here, creating 2 new actions that are specialized, similar to the JS example above.

### Computed Properties

CPs are pretty cool. They're declarative, so you can specify what a value should be when its dependent values change, much like a spreadsheet. This is in contrast to the imperative form where you would have to manually listen for changes in each e.g. input and then add the values together a la how you would do it in jQuery.

The most basic example of a CP is to to join a first and last name. Everyone has probably written a CP like this one.

```js
import Ember from 'ember';

const { Component, computed, get } = Ember; 

export default Component.extend({
  firstName: 'Jim',
  lastName: 'Bob',

  fullName: computed('firstName', 'lastName', function() {
    return `${get(this, 'firstName')} ${get(this, 'lastName')}`;
  }).readOnly()
});
```

Pretty simple! But how would you make something like this reusable? 

#### Computed property macros

Enter functional programming and computed property macros:

```js
import Ember from 'ember';

const { computed, get } = Ember;

export default function joinWith(separator, ...dependentKeys) {
  return computed(...dependentKeys, function() {
    return dependentKeys
      .map((dependentKey) => get(this, dependentKey))
      .join(separator);
  });
}
```

Let's talk about what's going on here. A computed property macro is a higher order function - it's a function that returns a function (much like the `pipe` function we wrote earlier).

The first line of the function says that the `joinWith` function is variadic - it takes in a "seperator" as the first argument, and any number of dependent keys as the rest. A "dependent key" is just a string which is the name of the key you want observed in the computed property. In the example of our `fullName` computed property, the dependent keys would be `firstName` and `lastName`.

Next, we return a new function, which happens to be a computed property! Here, we apply the array of dependent keys as arguments, which is the equivalent of us writing `computed('firstName', 'lastName', function() { /* ... */ })` except that this is dynamic at run time. 

Now we have an array of dependent keys: `['firstName', 'lastName']`. We `map` over this array, and return the value of `get(this, dependentKey)` which essentially is the same as writing `[get(this, 'firstName'), get(this, 'lastName')] === ['Jim', 'Bob']`.

Since `map` returns an array, we can now complete the function by chaining the `join` method. `join` joins an array together with a separator and returns a single string of the combined values. 

Now, we can use this macro in multiple places without repeating ourselves:

```js
import Ember from 'ember';
import joinWith from 'path/to/join-with';

const { Component } = Ember;

export default Component.extend({
  title: 'Mr',
  firstName: 'Jim',
  lastName: 'Bob',

  fullName: joinWith(' ', 'firstName', 'lastName'),
  fullNameWithTitle: joinWith(' ', 'title', 'firstName', 'lastName'),
  greetingName: joinWith(' ', 'title', 'lastName')
});
```

Cool! And you can pretty much do the exact same thing with any CP you have in your app that you are repeating in a bunch of places. Later we'll talk a little bit about how you can test CPs as well as macros.

### Helpers

Helpers are a pretty cool. You can essentially do functional programming in the template with these helpers, but you shouldn't get too carried away.

Here's a very basic example:

```js
import Ember from 'ember';

const { Helper: { helper } } = Ember;

export function sum(head, ...tail) {
  return tail.reduce((acc, curr) => acc + curr, head);
}

export default helper((values = []) => sum(...values));
```

Again, this is our very handy `reduce` method at work. The pattern here is similar to the `pipe` function we wrote earlier.

```hbs
{{sum 1 2 3}} <!-- 6 -->
```

A function helper in Ember is essentially a wrapped function. In the above example, I'm making a function called sum that takes a list of values and sums them together.

The important distinction to note here is that the Ember helper passes the values from the template as an array! What that means is that when we `export default helper(...)`, you'll notice that I explicitly spread the array of values into the sum function. 

This is slightly different from what is generated by ember-cli. The reason I'm writing it this way is that it promotes greater reusability. Since I'm exporting the function `sum`, I can import it elsewhere and use it like any old function.

This is the "conventional" way generated by ember-cli:

```js
import Ember from 'ember';

const { Helper: { helper } } = Ember;

export function sum(values = []) {
  return values.reduce((acc, curr) => acc + curr, 0);
}

export default helper(sum);
```

But here since `sum` expects a single array as an argument, it's very unlike how you would write a regular function in JavaScript. The reason why Ember does it this way is so that you can also receive an options hash in your helper:

```js
import Ember from 'ember';

const { Helper: { helper } } = Ember;

export function sum(values = [], options = {}) {
  // options = { someOption: true }
}

export default helper(sum);
```

```hbs
{{sum 1 2 3 someOption=true}} <!-- 6 -->
```

Helpers are great and very functional. For example, you can write a helper that returns a new function – this essentially lets you create new actions in Ember.

Show [pipe helper](https://github.com/DockYard/ember-composable-helpers/blob/master/addon/helpers/pipe.js) source as example.

This means that you can use it in your template like so:

```hbs
<button {{action (pipe addToCart purchase redirectToThankYouPage) item}}>
  1-Click Buy
</button>
```

Again, higher order functions at work!

Helpers can also be class based:

```js
export default Helper.extend({
  localesService: inject.service('locales'),
  currentLocale: readOnly('localesService.currentLocale'),
  
  compute([key]) {
    let currentLocale = get(this, 'currentLocale');
    
    return get(this, 'localesService').lookup(currentLocale, key);
  },

  localeDidChange: observer('currentLocale', function() {
    this.recompute();
  })
});
```

This means you can do anything you can do in an ordinary Ember Object. If you're not aware, almost every framework class in Ember inherits from Ember.Object. 

The class based helper expects a method called `compute` which takes in a similar argument signature to the function helper. You can also manually trigger recomputes using the `recompute` method, and use observers, etc.

## Part 2 - Testing

### Unit tests

Unit tests are the simplest kind of tests in Ember. They are typically run in isolation - meaning that no other parts of your app are involved (unless explicitly needed, e.g. services). Your app won't be rendered in this test, meaning you can't test your UI either.

Unit tests are best when you need to test some public function or method in your app against a variety of test cases. For example, let's write a unit test for the `sum` helper we wrote earlier.

#### Anatomy of a unit test

```js
import { sum } from 'path/to/sum';
import { module, test } from 'qunit';

module('Unit | Helper | sum');

test('it works', function(assert) {
  // ...
});
```

This is how a very basic helper test looks like. It imports the `module` and `test` functions from `qunit`, as well as imports the function `sum` from our helper.

The `module` function is just a way to group our tests together with a logical name. 

The `test` function is what we use to create a new test within this module. Let's write our first test.

#### Testing philosophy

A good unit test should always cover a variety of test cases, meaning we need to test both the happy and unhappy paths to using this function.

```js
test('it sums values together', function(assert) {
  // ...
});
```

The `test` function takes 2 arguments - the first is the name of the test, and the next is callback function that has an `assert` argument. We'll see that `assert` is just an object with some [useful methods](https://api.qunitjs.com/category/assert/) on it. Let's continue writing the happy path for our test:

```js
test('it sums values together', function(assert) {
  assert.equal(sum(1, 2, 3), 6);
  assert.equal(sum(3, 3, 3), 9);
  assert.equal(sum(0, 0, 0), 0);
});
```

Here, we're going to use the `equal` method in our assertions. The first argument is the result, and the next argument is what we expect the result to be. It's the equivalent of saying in pseudo-code that all 3 assertions are true:

```
sum(1, 2, 3) === 6; // true
sum(3, 3, 3) === 9; // true
sum(0, 0, 0) === 0; // true
```

We can even throw in a `notEqual`, which is self-explanatory:

```js
test('it sums values together', function(assert) {
  assert.notEqual(sum(1, 2, 3), 0);
});
```

Now we should test the unhappy cases. This is where you start thinking about the possible values that someone might pass to your function. For example, what if I accidentally passed in a `null` or `undefined`? Should the function warn the developer, or just ignore non-number values? This is when you start thinking about potential use-cases for your function and how you might want to handle them.

For example, let's say that in our scenario we want to ignore all non-numerical values:

```js
test('it ignores non-numerical values', function(assert) {
  assert.equal(sum(1, 2, null), 3);
  assert.equal(sum(1, 2, undefined), 3);
  assert.equal(sum(1, 2, 'foo'), 3);
});
```

#### Basic TDD

If we run this test now, it will fail since we have not handled this in our function. Let's update it:

```js
import Ember from 'ember';

const { Helper: { helper }, isBlank } = Ember;

export function sum(head, ...tail) {
  return tail
    .reject(isBlank)
    .reduce((acc, curr) => acc + curr, head);
}

export default helper((values = []) => sum(...values));
```

Here, we added `reject(isBlank)` to the sum function prior to `reduce`. The `reject` method will run the `isBlank` function on each item in the array, and if `true`, it removes it. In other words, only values that are not `null`, `undefined`, or empty strings and arrays will be passed along to our sum.

However, when we run our test again, we still see one failure. That's because we haven't really fixed our function, we have added the wrong logic.

If we ONLY want to sum numbers, that's what we should do - rejecting blank values only happens to handle part of this logic. Instead, we should do it like this:

```js
import Ember from 'ember';

const { Helper: { helper }, typeOf } = Ember;

const isNumber = (value) => typeOf(value) === 'number';

export function sum(head, ...tail) {
  return tail
    .filter(isNumber)
    .reduce((acc, curr) => acc + curr, head);
}

export default helper((values = []) => sum(...values));
```

Now all our test cases should pass. 

#### Data-driven testing

Something interesting to note about tests is that you can take a data driven approach to writing them - there is nothing special about a test. It's just JavaScript. What this means is that we can parameterize our happy and unhappy cases:

```js
import { sum } from 'path/to/sum';
import { module, test } from 'qunit';

module('Unit | Helper | sum');

const testData = [
  { data: [1, 2, 3], expected: 6 },
  { data: [3, 3, 3], expected: 9 },
  { data: [0, 0, 0], expected: 0 },
  { data: [1, 2, null], expected: 3 },  
  { data: [1, 2, undefined], expected: 3 },  
  { data: [1, 2, 'foo'], expected: 3 }  
];

testData.forEach(({ data, expected }) => function() {
  test('it sums numbers together', function(assert) {
    assert.equal(sum(...data), expected);
  });
});
```

#### Testing computed property macros (or anything that requires an `Ember.Object`)

So that's how you unit test a function. Earlier, we spoke about computed property macros. Let's see how you would test the `joinWith` macro we wrote earlier.

```js
import Ember from 'ember';

const { computed, get } = Ember;

export default function joinWith(separator, ...dependentKeys) {
  return computed(...dependentKeys, function() {
    return dependentKeys
      .map((dependentKey) => get(this, dependentKey))
      .join(separator);
  });
}
```

The first thing to note here is that since this is a computed property macro, we can't just test it like we would an ordinary function. This is because CPs are specific to `Ember.Object` which uses the `Ember.Observable` mixin. This essentially allows the KVO functionality inside of a special object. 

What this means is that we need an `Ember.Object` to test our CPM against. 

```js
import Ember from 'ember';
import joinWith from 'path/to/join-with';
import { module, test } from 'qunit';

const { Object: EmberObject, get } = Ember;

module('Unit | Utility | macros/join with');

test('#joinWith returns a string of values joined with a separator', function(assert) {
  let Employee = EmberObject.extend({
    fullName: joinWith(' ', 'firstName', 'lastName')
  });
  let subject = Employee.create({ firstName: 'Bill', lastName: 'Lumbergh' });

  assert.equal(get(subject, 'fullName'), 'Bill Lumbergh');
});

```

In the above example, we first created a new "class" that extends from a basic `Ember.Object`. This is denoted by the use of the capital first letter in `Employee`. Inside of this class, we can now use concepts we're familiar with, like injecting services, using CPs and so forth.

In our test, we then instantiate an instance of the `Employee` class, giving it a first and last name. When we then `get` the `fullName` CP from `subject`, it returns the result we we expect it to.

You can also unit test things like models, controllers and components - these will be tested without any UI, and allows you to unit test specific public methods.

### Integration tests

Unit tests are great for testing specific pieces of functionality in isolation, and they fit very well within the FP paradigm. 

However in Ember, we are often concerned with testing pieces of our UI. Let's see how we can do that with component integration tests. One key thing to note is that you can also integration test helpers. Also, your entire app (minus the router) is booted, so other components and helpers can be used within your test.

Here's a very basic example:

```js
import Ember from 'ember';
import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

const { RSVP: { resolve }, run } = Ember;

moduleForComponent('add-numbers', 'Integration | Component | {{add-numbers}}', {
  integration: true
});

test('it renders', function(assert) {
  this.set('x', '3');
  this.set('y', '3');
  this.on('someAction', (x, y) => x + y);
  this.render(hbs`
    {{add-numbers x=x y=y someAction=someAction}}
  `);

  assert.equal(this.$('p').text().trim(), '0', 'precond - should render 0');
  this.$('button').click();
  assert.equal(this.$('p').text().trim(), '6', 'should render 6');
});
```

The important thing to note about integration tests is how they differ from a unit test. Here, we need to `set` values and actions on the `test` context.

#### Why do we set values and actions on the test context?

Normally, when you use a component in Ember, the "context" is the template's controller from which the component is rendered. For example:

```hbs
// templates/application.hbs
{{foo-bar myValue=myValue}}
```

```js
// controllers/application.js
import Ember from 'ember';

const { Controller } = Ember;

export default Controller.extend({
  myValue: 'Jim Bob'
});
```

The value `myValue` is passed along from the template's controller - the application controller. 

Note that when a component is rendered inside of another component's template, that child component's controller is the parent component.

So in our integration test, we render the component within the test's template - so, the component's controller is the test itself! This means that in order to pass down values, we need to set it on the test "controller" itself. To do this, all you need to use is `this.set` for values and `this.on` for actions.

#### `this.$()`

Another thing to note is that inside of an integration test, in order to access the DOM, you have to use `this.$(selector)`, where `selector` is an optional string selector similar to how you would select an element using jQuery.

Since `this.$()` returns a jQuery-like object, you can use familiar things like the `click` event listener and so forth to trigger actions. Inside of an integration test you are essentially programmatically interacting with your UI - so try and avoid to do "unit testing" inside of it. What that means is, don't do things like get the container to access methods and so on. 

If your component is hard to integration test, it is a sign that your component is not very well written and probably needs to be refactored.

This is especially true when you are testing complex components that have child components that require a lot of data. This means that you'll end up having to write a lot of boilerplate in order to setup your component for testing.

Luckily for us, there is a technique we can use to decouple our components from one another - dependency injection.

#### Dependency injection in components

Let's say you have a complex component for editing a geolocation that has multiple child components - a map, form, sidebar and so on.

```hbs
<!-- templates/application.hbs -->

{{edit-location location=location}}
```

```hbs
<!-- components/edit-location.hbs -->

{{google-map 
    lat=location.lat 
    lng=location.lng 
    setLatLng=(action "setLatLng")
    setMarkerRadius=(action "setMarkerRadius")
}}
{{edit-location-form location=location}}
{{location-activity location=location}}
```

If you try to test this `edit-location` component now, you'll probably have a difficult time trying to set everything up so that it can even be rendered.

Thankfully, we have DI. DI is a fancy term for passing in dependencies as arguments instead of using them implicitly in our objects. Here, we're using these child components directly in the parent component's template, so we cannot "stub" them out in tests or even pass in other components to use, leading to repetition if we have subtly different variants of the `edit-location` component.

Component DI is actually very simple. You can pass in a component like you would any other argument, with the minor execption that it should be wrapped with a `hash`:

```hbs
<!-- application.hbs -->

{{edit-location 
    location=location
    ui=(hash
      location-map=(component "google-map")
      location-form=(component "edit-location-form")
      location-activity=(component "location-activity"))
}}
```

Here, we've wrapped our child components in a `ui` object (the name is not significant, I just chose something short and simple, but you can name it whatever you want). The `hash` helper in Ember basically creates an object on the fly.

Now, inside of our parent component, we can use the components as if they were oridinary arguments:

```hbs
<!-- components/edit-location.hbs -->

{{ui.location-map 
    lat=location.lat 
    lng=location.lng 
    setLatLng=(action "setLatLng")
    setMarkerRadius=(action "setMarkerRadius")
}}
{{ui.location-form location=location}}
{{ui.location-activity location=location}}
```

Cool! Pretty simple refactor.

Now, in our parent component integration test, we can register "dummy components" that do nothing that can be used in place of our child components.

First, install the [`ember-test-component`](https://github.com/poteto/ember-test-component) addon:

```
ember install ember-test-component
```

Then, we need a small bit of setup:

```js
import { registerTestComponent, unregisterTestComponent } from 'my-app/tests/ember-test-component';

moduleForComponent('...', {
  integration: true,
  beforeEach({ test: testCtx }) {
    unregisterTestComponent(testCtx.testEnvironment);
  }
});
```

This ensures that our `test-component` doesn't leak to other tests. Now, in our integration test, we can register test components on the fly:

```js
test('it does something', function(assert) {
  registerTestComponent(this);
  this.render(hbs`
    {{edit-location ui=(hash
        location-map=(component "test-component")
        location-form=(component "test-component")
        location-activity=(component "test-component"))
    }}
  `);
  // ...
});
```

What this means is that we can write our components to be better decoupled from one another. This means that we can test our child components in isolation as well, leading to clearer tests and cleaner code.

### Acceptance tests

Now finally, we get to acceptance tests. Acceptance tests test your entire application including routing. Because they tend to be run quite slowly, I don't write very many acceptance tests, instead relying more on component integration tests. However, acceptance tests are great for regression testing (tests that ensure bugs don't regress after being fixed) and making sure key app flows work as intended.

Here's a basic acceptance test:

```js
import { test } from 'qunit';
import moduleForAcceptance from 'people/tests/helpers/module-for-acceptance';

moduleForAcceptance('Acceptance | login');

test('visiting /login', function(assert) {
  visit('/login');

  andThen(() => assert.equal(currentURL(), '/login'));
});
```

Acceptance tests don't look very different, but the thing to note is that you get a bunch of [test helpers](https://guides.emberjs.com/v2.9.0/testing/acceptance/#toc_test-helpers) that you can use to interact with your application. 

This is in contrast to integration tests which _do not_ have these helpers - instead you interact with the DOM via `this.$()` which returns a jQuery-like object.

Another important thing to note is that some interactions are going to be async. This means that you often have to wrap interactions within an `andThen` function. This helper will wait for any pending promises to be resolved before being run.

For example, let's say that you have a button that saves your model. Saving is async, so we need to wait for the save promise to be resolved before we can see any updates to the UI:

```js
import { test } from 'qunit';
import moduleForAcceptance from 'people/tests/helpers/module-for-acceptance';

moduleForAcceptance('Acceptance | login');

test('visiting /login', function(assert) {
  visit('/login');

  andThen(() => assert.equal(currentURL(), '/login'));
  andThen(() => click('button'));
  andThen(() => assert.equal(find('h1').text().trim()), 'Jim Bob');
});
```

The `find` helper is similar to `this.$()` - it also returns a jQuery-like object.

#### Mocking API responses

In your acceptance tests, it is advisable / recommended that you also mock your API responses. This means that your app won't actually perform any real HTTP requests. Instead, you might use something like `pretender` or `mirage` to create fake servers that return canned JSON responses.

I personally dislike `ember-cli-mirage` as it is often too heavy of a solution. However you might disagree (which is completely fine) so if you wish to use it you can check it out [here](http://www.ember-cli-mirage.com/).

Under the hood, `ember-cli-mirage` uses Pretender as well. So let's see how Pretender works. 

First, install the `ember-cli-pretender` addon:

```
ember install ember-cli-pretender
```

This imports `pretender` into our tests, so we can import it and use it to create a fake API. `pretender` has an `Express`-like syntax for creating a fake server.

Let's set up a basic example:

```js
import Pretender from 'pretender';
import { test } from 'qunit';

import moduleForAcceptance from 'orion-ui/tests/helpers/module-for-acceptance';
import sendResponse from 'orion-ui/tests/helpers/send-response';
import me from '../../helpers/fixtures/me';

const apiUrl = '/api/v1';

moduleForAcceptance('Acceptance | recent-updates', {
  beforeEach() {
    this.server = new Pretender(function() {
      this.get(`${apiUrl}/users/me`, function() {
        return sendResponse(me);
      });
    });
  },

  afterEach() {
    this.server.shutdown();
  }
});

test('it should render user profile', function(assert) {
  visit('/profile');
  andThen(() => assert.equal(find(testSelector('selector', 'user-name')).text(), 'Jim Bob'));
});
```

`sendResponse` is a simple test helper for returning responses. You can place it within your `tests/helpers` directory and then import it:

```js
const { stringify } = JSON;

export default function sendResponse(data, statusCode = 200, headers = { 'Content-Type': 'application/json' }) {
  return [statusCode, headers, stringify(data)];
}
```

What we've done here is setup a fake server with one route: `/api/v1/users/me`. This will intercept any HTTP requests to that URL and then call that function. In our case, we are returning a canned response, which is just JSON that is exported as an object:

```js
/* jshint ignore:start */
/* jscs:disable */
/* Fetched on Oct 5th 2016 */
export default {
  "data": {
    "id": "1",
    "type": "user",
    "attributes": {
      "first-name": "Jim",
      "last-name": "Bob",
      "email": "jim@bob.com",
      "profile-image": "http://www.jimbob.com/selfie.jpg"
    }
  }
}
/* jshint ignore:end */
/* jscs:enable */
```

You can place this file anywhere, but we tend to place it in `tests/helpers/fixtures`, and then you can import it like you would any other module. The reason you can't just save it as JSON is that JSON is not exported, so you cannot import it using ES2015 syntax. However since JSON is a valid JavaScript object, you can just export it directly by prefixing the JSON with `export default`.

This is just a basic example, but you should look at the [Pretender documentation](https://github.com/pretenderjs/pretender) for writing more advanced stuff. Try to keep it simple though! 

Here is something more advanced, where we need to allow fetching single responses by `id`:

```js
import Pretender from 'pretender';
import { test } from 'qunit';

import JaQuery from 'orion-ui/tests/ember-ja-query';
import sendResponse from '../../../helpers/send-response';
import buyingTeamTypes from '../../../helpers/fixtures/buying-team-types';

const apiUrl = '/api/v1';

moduleForAcceptance('Acceptance | schedule/display/month', {
  beforeEach() {
    this.server = new Pretender(function() {
      this.get(`${apiUrl}/buying_team_types/:id`, function({ params }) {
        let { id } = params;
        let wrapped = new JaQuery(buyingTeamTypes);

        return sendResponse(wrapped.findBy('id', id));
      });
  },

  afterEach() {
    this.server.shutdown();
  }
});
```

In this example, we're using an addon called [`ember-ja-query`](https://github.com/poteto/ember-ja-query). This addon wraps a JSON-API response and then gives you query methods on top of it.

Above, we've wrapped an array response of JSON-API buying team types with `ja-query`. Instead of trying to traverse through the weird JSON-API schema, we can use `ja-query` methods to find a single record from that array and return it.

You can also use it to do other kinds of filtering and so forth when setting up your mock server.

## Part 3 - ember-concurrency

Dealing with concurrency and asynchrony when it comes to UI and JavaScript is never a fun exercise. It used to be done with deeply nested callbacks (aka callback hell). Thankfully, with modern JavaScript we have promises that make that slightly less terrible - however they are still a pain to deal with.

You may have heard of the new `async` and `await` syntax in JS that will let us deal with promises as if they were synchronous. [`ember-concurrency`](http://ember-concurrency.com/#/docs) is an addon that provides something similar, but using ES2015 generators instead. This is because `async/await` is still a stage 3 proposal in TC39 (ie it is not officially a part of the ES spec yet), and generator functions have similar and possibly even superior semantics since there is a mechanism for cancelation.

Let's look at a basic example of dealing with loading some async data in Ember today. We're probably familiar with this clever promise CP trick:

```js
import Ember from 'ember';

const { Component, computed } = Ember;

export default Component.extend({
  someAsyncProperty: computed('model.asyncData.propertyName', {
    get() {
      return get(this, 'model.asyncData.propertyName').then((prop) => {
        if (get(this, 'isDestroyed') || get(this, 'isDestroying')) {
          return;
        }
        
        // do stuff with `prop`
        set(this, 'someAsyncProperty', prop);
      });
    },
    set(_key, value) {
      return value;
    }
  })
});
```

What the above lets you do is to be able to `get` the `someAsyncProperty` key on the component, and have it set itself to the value of the resolved promise. Once a value is set on the CP, it then overwrites the getter so that subsequent `get`s will just retrieve the resolved value. 

However, it's obvious that the above is quite error-prone and involves a lot of defensive code that makes things hard to reason about and debug. With ember-concurrency, there is a simpler way:

```js
import Ember from 'ember';
import { task } from 'ember-concurrency';

const { Component, computed } = Ember;

export default Component.extend({
  _someAsyncProperty: task(function*() {
    yield get(this, 'model.asyncData.propertyName').then((prop) => {
      // do stuff with `prop`
      set(this, 'someAsyncProperty', prop);
    });
  }).on('init')
});
```

The above example is definitely a lot cleaner to read. `yield` is a special keyword that means to stop the generator function until the promise being yielded is resolved. The best part is, you can assign it to a value directly, as if it was async.

You'll note that I included the `on('init')` modifier, which as it suggests will run this function whenever a new component instance is created. You can also have the generator function run on different events using this same syntax.

Now let's see exactly what this `function*` / [generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators) is all about. 

In JavaScript, an `iterator` is:

> An object is an iterator when it knows how to access items from a collection one at a time, while keeping track of its current position within that sequence. In JavaScript an iterator is an object that provides a `next()` method which returns the next item in the sequence. This method returns an object with two properties: `done` and `value`.

```js
function makeIterator(array){
  let nextIndex = 0;
  
  return {
    next() {
      return nextIndex < array.length ?
        { value: array[nextIndex++], done: false } :
        { done: true };
    }
  }
}
```

```js
let it = makeIterator(['yo', 'ya']);
console.log(it.next().value); // 'yo'
console.log(it.next().value); // 'ya'
console.log(it.next().done);  // true
```

Generators are a simpler and less error-prone way of dealing with iterator state. Generators are special type of functions that work as a factory for iterators. A function is a generator if it contains at least one `yield` and if it also uses the `function*` syntax.

```js
function* idMaker() {
  let index = 0;
  while(true)
    yield index++;
}

let gen = idMaker();
console.log(gen.next().value); // 0
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
// ...
```

In this example, `while(true)` is an infinite loop. However because it wrapped in a generator, we can `yield` each iteration and only deal with values one at a time.

`ember-concurrency` takes this basic concept and extends it further, giving you nice concurrency and async primitives to work with in Ember, including cancelation. 

Although generators are already a part of the ES2016 spec, they are not fully implemented by browsers and so a Babel polyfill is required. This polyfill uses the [`regenerator`](https://github.com/facebook/regenerator) polyfill written by Facebook and is automatically included when you install the `ember-concurrency` addon.

#### Task syntax

`ember-concurrency` has a few different ways of performing tasks. Right now you can think of these as "actions" that need to be invoked, but there is work going on in the addon to eventually support async CPs, so we will be able to clean up our code even more. For example, using async CPs you can just return the `yield` instead of having to set another value. This is coming very soon, which is great!

But for now, you can mostly get by with using the `on('init')` hook for most cases. When you do want to do some async or concurrent work though, you need to `perform` the task:

```js
{
  myGenerator: task(function*(interval) {
    set(this, 'status', 'Gimme one second...');
    yield timeout(interval);
    set(this, 'status', 'Gimme one more second...');
    yield timeout(interval);
    set(this, 'status', "Finished!");
  })  
}

get('myGenerator').perform(1000);
```

You can pass arguments to your generator functions like you would an ordinary function. Note the use of the `yield timeout` expressions - `timeout` is a tiny utility from `ember-concurrency` that lets you wait for a number of milliseconds before proceeding on. This can be very useful for certain scenarios like debouncing and so on.

When the generator is performed, the function is called and the `status` is set accordingly. You could call this `perform` from within an action, or you could even use the `perform` helper:

```hbs
<button {{action perform myGenerator 1000}}>
  Do it
</button>
```

#### Task concurrency and cancelation

Now you might be wondering what `ember-concurrency` can do with these primitives. The answer is that this addon will change the way you write your ember apps. Dealing with complex UI is made easier, and dealing with concurrent actions extends that even further.

At this point, we'll go directly to the docs to show examples. Perhaps open up an `ember-twiddle` to do some live coding as well. 

- Talk about [task modifiers](http://ember-concurrency.com/#/docs/task-concurrency)
- Talk about [cancelation](http://ember-concurrency.com/#/docs/cancelation)

#### Examples

[Examples](http://ember-concurrency.com/#/docs/examples)
