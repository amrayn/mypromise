<p align="center">
    <a href="https://github.com/amrayn/craftx">
      <img width="190px" src="https://github.com/amrayn/craftx/raw/master/assets/logo.png?" />
    </a>
    <p align="center">Carefully craft JSON and async functions</p>
</p>

<p align="center">
  <a aria-label="Build Status" href="https://travis-ci.org/amrayn/craftx">
    <img alt="" src="https://img.shields.io/travis/amrayn/craftx/master.svg?style=for-the-badge&labelColor=000000">
  </a>
  <a aria-label="NPM version" href="https://www.npmjs.com/package/craftx">
    <img alt="" src="https://img.shields.io/npm/v/craftx.svg?style=for-the-badge&labelColor=000000">
  </a>
  <a aria-label="License" href="https://github.com/amrayn/craftx/blob/master/LICENSE">
    <img alt="" src="https://img.shields.io/npm/l/craftx?style=for-the-badge&labelColor=000000">
  </a>
  <a aria-label="Donate" href="https://amrayn.com/donate">
    <img alt="" src="https://img.shields.io/static/v1?label=Donate&message=Web&color=purple&style=for-the-badge&labelColor=000000">
  </a>
</p>

# Installation

```bash
npm i craftx -S
# or
yarn add craftx
```

# Introduction

Craftx is a utility for promises. It provides two basic functionalities:

1. Craft a JSON object and resolve any promise before final JSON is produced
2. Craft a function that would allow you to pass promises as parameters without worrying whether they're yet fulfilled or not.

## 1. Craft a JSON
### TL;DR

What do you think output would be?
```javascript
const calcAge = async () => 65;

(async () => {
  console.log({
    age: calcAge(),
  })
})();
```
Answer is: `Object {age: Promise (resolved)}`

How do we solve this problem? Either by adding a lot of awaits for each promise, or just by simply:

```javascript
const { json } = require("craftx");

const calcAge = async () => 65;

(async () => {
  console.log(await json({
    age: calcAge(),
  }))
})();
```

Try it on [RunKit](https://npm.runkit.com/craftx)

### The Problem

Let's say you need to create a JSON

```javascript
{
  id: 1,
  name: 'John F. Kennedy',
  age: 45,
}
```

This is good as long as you are not using promises, but if you want to use promises like:

```javascript
const queryName = () => Promise.resolve('John F. Kennedy');
const calculateAge = () => Promise.resolve(45);

{
  id: 1,
  name: queryName(),
  age: calculateAge(),
}
```

this will result potentially unresolved promises.

and if you do this:

```javascript
{
  id: 1,
  name: await queryName(),
  age: await calculateAge(),
}
```

the calls are now sequential and defeats the purpose of [non-blocking event based I/O](https://developers.redhat.com/blog/2016/08/16/why-should-i-use-node-js-the-non-blocking-event-io-framework/) that Node.js is known for.

### Solution
To handle this situation without wrapping the promise resolution in a separate function, you can use this utility package to handle this situation

```javascript
const { json } = require('craftx');

await json({
  id: 1,
  name: queryName(),
  age: calculateAge(),
})
```

This will resolve only after all the promises are resolved. Resulting in:

```javascript
{
  id: 1,
  name: 'John F Kennedy',
  age: 45,
}
```

### Max depth
Default object depth supported by craftx is 64.

## 2. Craft a Function

### TL;DR

What do you think output would be?
```javascript
const getAge = async () => 123;
const getDetail = async (age) => `The age is ${age}`;

(async () => {
  const result = await getDetail(getAge());
  console.log(result);
})();
```
Answer is: `The age is [object Promise]`

How do we solve this problem?

```javascript
const craftx = require("craftx")

const getAge = async () => 123;
const getDetail = craftx.fn(async (age) => `The age is ${age}`);

(async () => {
  const result = await getDetail(getAge());
  console.log(result);
})();
```

Try it on [RunKit](https://npm.runkit.com/craftx)

### Problem
There are times when you want to use promise values without awaiting (i.e, automatically once the promise is fulfilled). This utility helps you achieve this goal using native promise mechanism.

We will walk you through an example and provide explanation where necessary.

Let's say you have various utility functions to query the database.

```javascript
const queryCompanyInfo_ = async () => ({
  username: '@amrayn',
});

const queryAccountInfo_ = async (company) => ({
  company,
  created: '19-02-2020',
});
```

Notice the `queryAccountInfo_` takes `company` parameter (promise) that will be provided by `queryCompanyInfo_`. But you don't know whether this promise is fulfilled or not. If you use `Promise.all` directly (without this library) you won't be able to provide this (resolved) company object to `queryAccountInfo_`.

```javascript
Promise.all([
  queryCompanyInfo_(),
  queryAccountInfo_(), // notice we cannot provide "company" here
]).then(([userInfo, accountInfo]) => {
  console.log(accountInfo);
})
```

A possible solution is to await for the promises first:

```javascript
const companyInfo = await queryCompanyInfo_();
const accountInfo = queryAccountInfo_(companyInfo);
```

This has 2 basic problems.

1. You're not making use of parallelism here. Which defeats the purpose of Promises to some extent.
2. The code is very soon going to be messy and unreadable.

### Solution
craftx allows you to "craft" a function that will help you achieve your goal without worrying about any of the above problem.

```javascript
const { fn } = require('craftx'); // or you can import { fn }

const queryCompanyInfo = fn(queryCompanyInfo_);
const queryAccountInfo = fn(queryAccountInfo_);

const finalJson = await queryAccountInfo(queryCompanyInfo())
```

This will result in:

```javascript
{
  company: {
    username: '@amrayn',
  },
  created: '19-02-2020',
}
```

## Misc

### Options
If the first parameter is an object for the `fn()`, that object is used for setting up the options.

For example:

```javascript
const getNumb = fn(() => 123, {
  startTime: res.startTime,
  endTime: res.endTime,
});
```

You can also override the options for a crafted function later.

```javascript
getNumb.setOptions({
  // hint: server-timing
  startTime: res.startTime,
  endTime: res.endTime,

  name: 'getNumb',
  description: 'Get number',

  debug: false, // to enable craftx debugging
});
```

Following are the possible options

| **Option** | **Description** |
|--|--|
| `name` | An identity for the function. Defaults to `<function>.name` - **IT MUST NOT CONTAIN SPACE** |
| `description` | A description for the function |
| `startTime` | Function for [server timing](https://www.w3.org/TR/server-timing/) - `(name, description) => {}` - the `name` and `description` is passed back to this function |
| `endTime` | Function for [server timing](https://www.w3.org/TR/server-timing/) - `(name) => {}` - the `name` is passed back to this function |
| `debug` | Boolean value to tell craftx whether debug logging is enabled or not. It will use a global `logger.debug()` object. If no such object exists, it will use `console.debug()` |

### Bulk Export
Converting existing exports to crafted functions is easy, either using `fn` for each function which can be cumbersome depending on number of functions; or you can simply convert the whole object using a helper function `fnExport`.

Let's say you have:

```javascript
const function1 = () => {}
const function2 = () => {}

module.exports = {
  function1,
  function2,
}
```
Just use `fnExport` when exporting

```javascript
const { fnExport } = require('craftx');

const function1 = () => {}
const function2 = () => {}

module.exports = fnExport({
  function1,
  function2,
})
```

Alternatively, you can do it when importing like in example of `/examples/json.js`. Doing it multiple times does not harm.

### Get Object Value
If you have a function that returns an object, and you want to grab just one specific value from the object, you can use built-in `get` function to do that.

```javascript
const { get } = require('../src');

const getProfile = async (uid) => ({
  name: 'John',
  age: 45,
  father: {
    name: 'Peter',
  },
});

(async () => {
  console.log(await get(getProfile(), 'father.name')) // output: Peter
  console.log(await get(getProfile(), 'mother.name', 'Steph')) // output: Steph
  console.log(await get(getProfile(), 'brother.name')) // output: undefined
})();
```

Synopsis: `get(object, path, defaultValue, options)`. The `options` is passed through to `fn()` internally.

**NOTE:** This function uses [lodash.get](https://www.npmjs.com/package/lodash.get) to get the path.

### Execute Directly
If you want to execute function directly without "crafting" a function, you can do so using `exec()`

```javascript
const { exec } = require('../src');
const { queryAccountInfo, queryUserInfo } = require('./example-utils');

(async () => {
  console.log(await exec(queryAccountInfo, queryUserInfo()));
})();
```

In above example, the resolved value from `queryUserInfo()` is guaranteed to be available in `queryAccountInfo(userInfo)`

Synopsis:
 * `exec(func, param1, param2, ...)`
 * `exec(options, func, param1, param2, ...)`

# License
```
Copyright (c) 2020-present Amrayn Web Services

https://github.com/amrayn/craftx
https://amrayn.com

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
