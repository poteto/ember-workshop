# ember-workshop

## Part 1 - Basics

### FP in JavaScript

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

Well, since you can pass functions around in JavaScript, let's see what we can do with the handy `reduce` method:

```js
const pipe = (fn, ...fns) => (...args) => fns.reduce((acc, f) => f(acc), fn(...args));
```

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

TODO
- [ ] Currying?

### CPs

CPs are pretty cool. They're declarative, so you can specify what a value should be when its dependent values change, much like a spreadsheet. This is in contrast to the imperative form where you would have to manually listen for changes in each e.g. input and then add the values together a la how you would do it in jQuery.

The most basic example of a CP is to to join a first and last name. Everyone has probably written a CP like this one.

```js
const Ember from 'ember';

const { Component, computed, get } = Ember; 

export default Component.extend({
  firstName: 'Jim',
  lastName: 'Bob',

  fullName: computed('firstName', 'lastName', function() {
    return `get(this, 'firstName') get(this, 'lastName')`;
  }).readOnly()
});
```

Pretty simple! But how would you make something like this reusable? Enter functional programming and computed property macros:

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
const Ember from 'ember';
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

If we run this test now, it will fail since we have not handled this in our function. Let's update it:

```js
import Ember from 'ember';

const { Helper: { helper }, isBlank } = Ember;

export function sum(head, ...tail) {
  return tail.reject(isBlank).reduce((acc, curr) => acc + curr, head);
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
  return tail.filter(isNumber).reduce((acc, curr) => acc + curr, head);
}

export default helper((values = []) => sum(...values));
```

Now all our test cases should pass. 

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

TODO
- [ ] needs

### Integration tests

TODO 
- [ ] Testing UI
- [ ] Using other helpers and components

### Acceptance tests

TODO
- [ ] Testing app flows
- [ ] Writing regression tests

## Part 3 - ember-concurrency

- [ ] Basic example of using the run loop
- [ ] Guarding against your component being destroyed
- [ ] Doing the same thing with ember-concurrency
- [ ] A little bit about promises
- [ ] What are generator functions?
- [ ] Task concurrency
- [ ] Cancelation 
