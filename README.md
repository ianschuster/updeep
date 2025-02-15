# updeep

> Easily update nested frozen objects and arrays in a declarative and immutable
> manner.

## This library is still maintained but is considered feature complete. Please feel free to use it and report any issues you have with it.

[![Join the chat at https://gitter.im/substantial/updeep](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/substantial/updeep?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![NPM version][npm-image]][npm-url]
[![Build Status][circleci-image]][circleci-url]
[![Code Climate][codeclimate-image]][codeclimate-url]

## About

updeep makes updating deeply nested objects/arrays painless by allowing you to
declare the updates you would like to make and it will take care of the rest. It
will recursively return the same instance if no changes have been made, making
it ideal for using reference equality checks to detect changes (like
[PureRenderMixin]).

Because of this, everything returned by updeep is frozen. Not only that, but
updeep assumes that every object passed in to update is immutable, so it may
freeze objects passed in as well. Note that the freezing only happens in
development.

updeep requires [lodash], but works very well with [lodash/fp] or [Ramda].
As a matter of fact, many of the helpers functions are [curried][currying]
[lodash] functions with their parameters reversed (like [lodash/fp]).

Note that the parameters may be backwards from what you may be used to. updeep
supports [partial application][currying], so the parameter order is:
`updeep(updates, object)`.

## API and Examples

### Full example

```js
const u = require('updeep');

const person = {
  name: { first: 'Bill', last: 'Sagat' },
  children: [
    { name: 'Mary-Kate', age: 7 },
    { name: 'Ashley', age: 7 }
  ],
  todo: [
    'Be funny',
    'Manage household'
  ],
  email: 'bill@example.com',
  version: 1
};

const inc = function(i) { return i + 1; }
const eq = function(x) { return function(y) { return x == y } };

const newPerson = u({
  // Change first name
  name: { first: 'Bob' },
  // Increment all children's ages
  children: u.map({ age: inc }),
  // Update email
  email: 'bob@example.com',
  // Remove todo
  todo: u.reject(eq('Be funny')),
  // Increment version
  version: inc
}, person);
// => {
//  name: { first: 'Bob', last: 'Sagat' },
//  children: [
//    { name: 'Mary-Kate', age: 8 },
//    { name: 'Ashley', age: 8 }
//  ],
//  todo: [
//    'Manage household'
//  ],
//  email: 'bob@example.com',
//  version: 2
//}
```

**NOTE**: All functions are curried, so if you see `f(x(, y))`, it can be called with either `f(x, y)` or `f(x)(y)`.

### `u(updates(, object))`

Update as many values as you want, as deeply as you want. The `updates` parameter can either be an object, a function, or a value. Everything returned from `u` is frozen recursively.

If `updates` is an object, for each key/value, it will apply the updates specified in the value to `object[key]`.

If `updates` is a function, it will call the function with `object` and return the value.

If `updates` is a value, it will return that value.

Sometimes, you may want to set an entire object to a property, or a function. In that case, you'll need to use a function to return that value, otherwise it would be interpreted as an update. Ex. `function() { return { a: 0 }; }`.

Also available at `u.update(...)`.

#### Simple update
Object properties:

```js
const person = {
  name: {
    first: 'Jane',
    last: 'West'
  }
};

const result = u({ name: { first: 'Susan' } }, person);

expect(result).to.eql({ name: { first: 'Susan', last: 'West' } });
```

Array elements:

```js
const scoreboard = {
  scores: [12, 28]
};

const result = u({ scores: { 1: 36 } }, scoreboard);

expect(result).to.eql({ scores: [12, 36] });
```

#### Multiple updates

```js
const person = {
  name: {
    first: 'Mike',
    last: 'Smith'
  },
  scores: [12, 28]
};

const result = u({ name: { last: 'Jones' }, scores: { 1: 36 } }, person);

expect(result).to.eql({ name: { first: 'Mike', last: 'Jones' }, scores: [12, 36] });
```

#### Use a function

```js
function increment(i) { return i + 1; }
const scoreboard = {
  scores: {
    team1: 0,
    team2: 0
  }
};

const result = u({ scores: { team2: increment } }, scoreboard);

expect(result).to.eql({ scores: { team1: 0, team2: 1 } });
```

#### Array Manipulation

Non-trivial array manipulations, such as element removal/insertion/sorting, can be implemented with functions. Because there are so many possible manipulations, we don't provide any helpers and leave this up to you. Simply ensure your function is pure and does not mutate its arguments.

```js
function addTodo(todos) { return [].concat(todos, [{ done: false }]); }

const state = {
  todos: [
    { done: false },
    { done: false }
  ]
};

const result = u({ todos: addTodo }, state);

expect(result).to.eql({ todos: [{ done: false }, { done: false }, { done: false }]});
```

[lodash/fp] is one of the many libraries providing good utility functions for
such manipulations.

```js
import fp from 'lodash/fp';

let state = {
  todos: [
    { done: true },
    { done: false }
  ]
};

// add a new todo
state = u({ todos: fp.concat({ done: false }) }, state);
expect(state).to.eql({ todos: [{ done: true }, { done: false }, { done: false }]});

// remove all done todos
state = u({ todos: fp.reject({ done: true }) }, state);
expect(state).to.eql({ todos: [{ done: false }, { done: false }]});
```

#### When null or undefined object, updeep uses a default object

```javascript
const result = u({ foo: 'bar' }, null);
expect(result).to.eql({ foo: 'bar' });
```

#### Partial application

```js
function increment(i) { return i + 1; }

const addOneYear = u({ age: increment });
const result = addOneYear({ name: 'Shannon Barnes', age: 62 });

expect(result).to.eql({ name: 'Shannon Barnes', age: 63 });
```

#### ES6 computed properties

```js
const key = 'age';

const result = u({ person: { [key]: 21 } }, { person: { name: 'Olivier P.', age: 20 } });

expect(result).to.eql({ person: { name: 'Olivier P.', age: 21 } });
```

### `u.freeze`

Freeze your initial state to protect against mutations. Only performs the freezing in development, and returns the original object unchanged in production.

```js
const state = u.freeze({ someKey: "Some Value" })
state.someKey = "Mutate" // ERROR in development
```

### `u._`

All updeep functions are curried.
If you want to partially apply a function in an order other than the default argument order, you can use the placeholder.

```js
function increment(i) { return i + 1; }
const updateJoe = u(u._, { name: "Joe Merrill", age: 21 });
const result = updateJoe({ age: increment });

expect(result).to.eql({ name: "Joe Merrill", age: 22 });
```

### `u.updateIn(path(, value)(, object))`

Update a single value with a simple string or array path. Can be use to update nested objects, arrays, or a combination. Can also be used to update every element of a nested array with `'*'`.

```js
const result = u.updateIn('bunny.color', 'brown', { bunny: { color: 'black' } });

expect(result).to.eql({ bunny: { color: 'brown' } });
```

```js
const result = u.updateIn('0.1.color', 'brown', [[{ color: 'blue' }, { color: 'red' }], []]);

expect(result).to.eql( [[{ color: 'blue' }, { color: 'brown' }], []]);
```

```js
function increment(i) { return i + 1; }

const result = u.updateIn('bunny.age', increment, { bunny: { age: 2 } });

expect(result).to.eql({ bunny: { age: 3 } });
```

```js
const result = u({ pets: u.updateIn([0, 'bunny', 'age'], 3) }, { pets: [{ bunny: { age: 2 } }] });

expect(result).to.eql({ pets: [{ bunny: { age: 3 } }] });
```

```js
const result = u.updateIn('todos.*.done', true, {
  todos: [
    { done: false },
    { done: false },
  ]
});

expect(result).to.eql({
  todos: [
    { done: true },
    { done: true },
  ]
});
```

### `u.constant(object)`

Sometimes, you want to replace an object outright rather than merging it.
You'll need to use a function that returns the new object.
`u.constant` creates that function for you.

```js
const user = {
  name: 'Mitch',
  favorites: {
    band: 'Nirvana',
    movie: 'The Matrix'
  }
};

const newFavorites = {
  band: 'Coldplay'
};

const result = u({ favorites: u.constant(newFavorites) }, user);

expect(result).to.eql({ name: 'Mitch', favorites: { band: 'Coldplay' } });
```

```js
const alwaysFour = u.constant(4);
expect(alwaysFour(32)).to.eql(4);
```

### `u.if(predicate(, updates)(, object))`

Apply `updates` if `predicate` is truthy, or if `predicate` is a function.
It evaluates to truthy when called with `object`.

```js
function isEven(x) { return x % 2 === 0; }
function increment(x) { return x + 1; }

const result = u({ value: u.if(isEven, increment) }, { value: 2 });

expect(result).to.eql({ value: 3 });
```

### `u.ifElse(predicate(, trueUpdates)(, falseUpdates)(, object))`

Apply `trueUpdates` if `predicate` is truthy, or if `predicate` is a function.
It evaluates to truthy when called with `object`. Otherwise, apply `falseUpdates`.

```js
function isEven(x) { return x % 2 === 0; }
function increment(x) { return x + 1; }
function decrement(x) { return x - 1; }

const result = u({ value: u.ifElse(isEven, increment, decrement) }, { value: 3 });

expect(result).to.eql({ value: 2 });
```

### `u.map(iteratee(, object))`

If iteratee is a function, map it over the values in `object`.
If it is an object, apply it as updates to each value in `object`,
which is equivalent to  `u.map(u(...), object)`).

```js
function increment(x) { return x + 1; }

const result = u({ values: u.map(increment) }, { values: [0, 1] });

expect(result).to.eql({ values: [1, 2] });
```

```js
function increment(x) { return x + 1; }

const result = u.map(increment, [0, 1, 2]);

expect(result).to.eql([1, 2, 3]);
```

```js
function increment(x) { return x + 1; }

const result = u.map(increment, { a: 0, b: 1, c: 2 });

expect(result).to.eql({ a: 1, b: 2, c: 3 });
```

```js
const result = u.map({ a: 100 }, [{ a: 0 }, { a: 1 }]);

expect(result).to.eql([{ a: 100 }, { a: 100 }]);
```

### `u.omit(predicate(, object))`

Remove properties. See [`_.omit`](https://lodash.com/docs#omit).

```js
const user = { user: { email: 'john@aol.com', username: 'john123', authToken: '1211..' } };

const result = u({ user: u.omit('authToken') }, user);

expect(result).to.eql({ user: { email: 'john@aol.com', username: 'john123' } });
```

```js
const user = {
  user: {
    email: 'john@aol.com',
    username: 'john123',
    authToken: '1211..',
    SSN: 5551234
  }
};

const result = u({ user: u.omit(['authToken', 'SSN']) }, user);

expect(result).to.eql({ user: { email: 'john@aol.com', username: 'john123' } });
```

### `u.omitted`

A property updated to this constant will be removed from the final object.
Useful when one wishes to remove and update properties in a single operation.

```js
const user = { email: 'john@aol.com', username: 'john123', authToken: '1211..' };

const result = u({ authToken: u.omitted, active: true }, user);

expect(result).to.eql({ user: { email: 'john@aol.com', username: 'john123', active: true } });
```

### `u.omitBy(predicate(, object))`

Remove properties. See [`_.omitBy`](https://lodash.com/docs#omitBy).

```js
const user = {
  user: {
    email: 'john@aol.com',
    username: 'john123',
    authToken: '1211..',
    SSN: 5551234
  }
};

function isSensitive(value, key) { return key == 'SSN' }
const result = u({ user: u.omitBy(isSensitive) }, user);

expect(result).to.eql({ user: { email: 'john@aol.com', username: 'john123', authToken: '1211..' } });

```

### `u.reject(predicate(, object))`

Reject items from an array. See [`_.reject`](https://lodash.com/docs#reject).

```js
function isEven(i) { return i % 2 === 0; }

const result = u({ values: u.reject(isEven) }, { values: [1, 2, 3, 4] });

expect(result).to.eql({ values: [1, 3] });
```

### `u.withDefault(default(, updates)(, object))`

Like `u()`, but start with the default value if the original value is undefined.

```js
const result = u({ value: u.withDefault([], { 0: 3 }) }, {});

expect(result).to.eql({ value: [3] });
```

See the [tests] for more examples.

### `u.is(path(, predicate)(, object))`

Returns `true` if the `predicate` matches the `path` applied to the `object`.
If the `predicate` is a function, the result is returned. If not, they are compared with `===`.

```js
const result = u.is('friend.age', 22, { friend: { age: 22 } });

expect(result).to.eql(true);
```

```js
function isEven(i) { return i % 2 === 0; }

const result = u.is('friend.age', isEven, { friend: { age: 22 } });

expect(result).to.eql(true);
```

```js
const person = {
  person: {
    name: {
      first: 'Jen',
      last: 'Matthews'
    }
  }
};

// Update person's last name to Simpson if their first name is Jen
const result = u({
  person: u.if(
    u.is('name.first', 'Jen'),
    u.updateIn('name.last', 'Simpson')
  )
}, person);

expect(result).to.eql({ person: { name: { first: 'Jen', last: 'Simpson' } } });
```

## Install

```sh
$ npm install --save updeep
```

## Configuration

If `NODE_ENV` is `"production"`, updeep will not attempt to freeze objects.
This may yield a slight performance gain.

## Motivation

While creating reducers for use with [redux], I wanted something that made it
easy to work with frozen objects. Native javascript objects have some nice
advantages over things like [Immutable.js][immutablejs] such as debugging and
destructuring. I wanted something more powerful than [icepick] and more
composable than [React.addons.update].

If you're manipulating massive amounts of data frequently, you may want to
benchmark, as [Immutable.js][immutablejs] should be more efficient in that
case.

## Contributing

1. Fork it.
1. Create your feature branch (`git checkout -b my-new-feature`).
1. Run `yarn test` and `yarn lint` to run tests and lint.
1. Commit your changes (`git commit -am 'Some change is made`).
1. Push to the branch (`git push origin my-new-feature`).
1. Create new Pull Request.

## Releasing New Version

1. Login to npm, if you don't have access to the package, ask for it.

    ```bash
    $ npm login
    ```
1. Make sure the build passes (best to let it pass on circleci, but you can run it locally):

    ```bash
    $ npm build
    ```
1. Bump the version:

    ```bash
    $ npm version major|minor|patch
    ```

1. Update the `CHANGELOG.md`.
  1. Add the new version and corresponding notes.
  1. Add a link to the new version.
  1. Update the `unreleased` link compare to be based off of the new version.
1. Publish and push:

```bash
$ npm publish
$ git push origin master --follow-tags
```

## See Also

A [Dash][dash] version of the documention is also available
as part of their user-contributed docsets.
For [Zeal][zeal] users, the user-contributed docsets
can be accessed via [zealusercontributions.herokuapp.com][zealuc].

## License

MIT ©2015 [Substantial](http://substantial.com)

[zealuc]: http://zealusercontributions.herokuapp.com/
[dash]: https://kapeli.com/dash
[zeal]: https://zealdocs.org/
[docset]: https://github.com/yanick/dash-docset-updeep
[npm-image]: https://badge.fury.io/js/updeep.svg
[npm-url]: https://npmjs.org/package/updeep
[circleci-image]: https://circleci.com/gh/substantial/updeep.svg?style=shield
[circleci-url]: https://circleci.com/gh/substantial/updeep
[daviddm-image]: https://david-dm.org/substantial/updeep.svg?theme=shields.io
[daviddm-url]: https://david-dm.org/substantial/updeep
[daviddm-peer-image]: https://david-dm.org/substantial/updeep/peer-status.svg
[daviddm-peer-url]:https://david-dm.org/substantial/updeep#info=peerDependencies
[daviddm-dev-image]: https://david-dm.org/substantial/updeep/dev-status.svg
[daviddm-dev-url]:https://david-dm.org/substantial/updeep#info=devDependencies
[codeclimate-image]: https://codeclimate.com/github/substantial/updeep/badges/gpa.svg
[codeclimate-url]: https://codeclimate.com/github/substantial/updeep
[lodash]: http://lodash.com
[lodash/fp]: https://github.com/lodash/lodash/wiki/FP-Guide
[Ramda]: http://ramdajs.com/
[PureRenderMixin]: https://facebook.github.io/react/docs/pure-render-mixin.html
[redux]: https://github.com/gaearon/redux
[immutablejs]: https://github.com/facebook/immutable-js
[icepick]: https://github.com/aearly/icepick
[React.addons.update]: https://facebook.github.io/react/docs/update.html
[tests]: https://github.com/substantial/updeep/blob/master/test/index.js
[currying]: http://www.datchley.name/currying-vs-partial-application/
