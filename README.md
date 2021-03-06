[![Build Status](https://travis-ci.org/practicalmeteor/meteor-munit.svg?branch=master)](https://travis-ci.org/practicalmeteor/meteor-munit)

## Munit

**Munit** stands for meteor unit (tests). It is a wrapper around Tinytest, the package testing framework shipped with meteor. **Munit** adds support for test suites and some additional functionality that is standard in other testing frameworks, such as test timeouts, setup, tearDown, suiteSetup, and suiteTearDown.

For additional information regarding Tinytest, please refer to this excellent screencast from the guys at EventedMind: [Testing Packages with Tinytest](https://www.eventedmind.com/feed/meteor-testing-packages-with-tinytest
)

## Installation

``meteor add practicalmeteor:munit``

[practicalmeteor:chai](https://atmospherejs.com/practicalmeteor/chai) and [practicalmeteor:sinon](https://atmospherejs.com/practicalmeteor/sinon) will be automatically added as well, which you can use in your tests.

## BDD Interface

MUnit allows you to use `describe` and `it` declaration blocks to declare tests:

```javascript

describe('suite1', function(){
  beforeAll(function (test){
    // Let's do 'cleanup' beforeEach too, in case another suite didn't clean up properly
    spies.restoreAll();
    stubs.restoreAll();
    console.log("I'm beforeAll");
  });
  beforeEach(function (test){
    console.log("I'm beforeEach");
    spies.create('log', console, 'log');
  });
  afterEach(function (test){
    spies.restoreAll();
    console.log("I'm afterEach");
  });
  afterAll(function (test){
    console.log("I'm afterAll");
    spies.restoreAll();
    stubs.restoreAll();
  });
  it('test1', function(test){
    console.log('Hello world');
    expect(spies.log).to.have.been.calledWith('Hello world');
  })
});
```

### The test argument

The `test` argument is the same test object passed to a test function by `Tinytest.add`, and has the following methods:

* `equal(actual, expected, msg)`
* `notEqual(actual, expected, msg)`
* `instanceOf(obj, klass)`
* `matches(actual, regexp, msg)`
* `throws(func, expected)`
* `isTrue(value, msg)`
* `isFalse(value, msg)`
* `isNull(value, msg)`
* `isNotNull(value, msg)`
* `isUndefined(value, msg)`
* `isNaN(value, msg)`
* `include(object, key)`
* `include(string, substring)`
* `include(array, value)`
* `length(obj, expected_length, msg)`

The `msg` property is a custom error message for the assertion.

You can use either the test object for your assertions or the included practicalmeteor:chai library.

You can see the source code [here](https://github.com/meteor/meteor/blob/devel/packages/tinytest/tinytest.js).

### Asynchronous Tests

To run a test asynchronously, add a `waitFor` callback wrapper as an argument to your test function. When calling an async function, you need to wrap your callback with 'waitFor'. This will let MUnit know that a callback is pending and that the test will be done once the callback was called and has done it's thing.

```javascript

describe('suite2', function(){
  it('async test', function(test, waitFor){
    var onTimeout = function () {
      try {
        expect(true).to.be.true;
      } catch(err) {
        test.exception(err);
      }
    };
    Meteor.setTimeout(waitFor(onTimeout), 50);
  });
});
```

#### **IMPORTANT NOTES**

1. As in any testing framework, you must enclose your async callback code in a try catch block, and report any exceptions to the framework, otherwise the framework has no way of knowing that exceptions occurred in async code. In the case of munit and tinytest, you report those exceptions using test.exception, as seen above.

2. Unfortunately, you cannot have more than one async function call per test. This is a limitation of the testAsyncMulti function from the meteor test-helpers core package that MUnit uses to run your test's beforeEach, test, and afterEach as part of the same test. We hope to eliminate this limitation soon.

3. beforeAll, beforeEach, afterAll, and afterEach can also be asynchronous, using the same test and waitFor arguments.

### Nested Describes

```javascript

describe('top-level describe', function(){
  describe('nested describe', function() {
    describe('deep nested describe', function() {
      it('a test', function () {
        expect(true).to.be.true;
      })
    })
  })
});
```

### Skipping Tests & Test Suites

```javascript

// Skipped suite
describe.skip('skipped suite', function(){
  it('work in progress', function(){
    expect(false).to.be.true;
  });
});

// Skipped test
describe('suite with skipped test', function(){
  it.skip('skipped test', function(){
    expect(false).to.be.true;
  });
});
```

### Server & Client Only Tests & Test Suites

Server & Client Only Tests:

```javascript

describe('client only and server only tests', function(){
  it.client('runs only in client', function(){
    expect(Meteor.isClient).to.be.true;
  });
  it.server('runs only in server', function(){
    expect(Meteor.isServer).to.be.true;
  });
});
```

Server & Client Only Test Suites:

```javascript

describe.client('client only test suite', function(){
  it('runs only in client', function(){
    expect(Meteor.isClient).to.be.true;
  });
  it.server('overrides describe.client and runs only in server', function(){
    expect(Meteor.isServer).to.be.true;
  });
});

describe.server('server only test suite', function(){
  it('runs only in server', function(){
    expect(Meteor.isServer).to.be.true;
  });
  it.client('overrides describe.server and runs only in client', function(){
    expect(Meteor.isClient).to.be.true;
  });
});
```

## TDD Interface

The TDD interface defers from the BDD interface in that it allows you to specify timeouts per test.

Create a JavaScript object or CoffeeScript class, with the following properties:

* `name`: String. The test suite name, with support for dashes for sub-grouping, as in Tinytest
* `suiteSetup`: Function. Runs once before all tests.
* `suiteTearDown`: Function. Runs once after all tests.
* `setup`: Function. Runs before each test.
* `tearDown`: Function. Runs after each test.
* `test<Name>` Function. Any function prefixed with `test` will run as a test case.
* `clientTest<Name>` Same as above, but will only run in the browser.
* `serverTest<Name>` Same as above, but will only run on the server.
* `tests`: In addition to test functions that start with 'test', you can provide an array of test objects, with additional fine tuning and control options.

Each test object in the `tests` array, can have the following properties:

* `name`: the name of the test case (**required**)
* `func`: the test case function (**required**)
* `type`: where to run the tests, either `client` or `server`. By default, runs in both.
* `timeout`: test timeout, in milliseconds (**default 5000**)
* `skip`: skip the test

To run your test suite, just:

```javascript

Munit.run( yourTestSuiteObject );
```

### Synchronous Tests

```javascript

mySyncSuite = {
  testSyncTest: function(test){
    test.isTrue(true);
  }
}

Munit.run(mySuite);
```

### Asynchronous Tests

```javascript

myAsyncSuite = {
  name: 'myAsyncSuite',

  testAsyncTest: function(test, waitFor){

    var onTimeout = function(){
      test.isTrue(true);
    }

    Meteor.setTimeout(waitFor(onTimeout), 50);
  }
};

Munit.run(myAsyncSuite);
```

The `waitFor` argument is the `expect` function wrapper passed to a test by testAsyncMulti from the meteor test-helpers package. In your test, you need to wrap your async callback function with waitFor, so testAsyncMulti knows that the test became asynchronous and a callback is pending. Unfortunately, you cannot have more than one async function call per test, due to the way testAsyncMulti works. We hope to eliminate this limitation soon.

### Complete Example

```javascript

tddTestSuite = {

  name: "TDD test suite",

  suiteSetup: function () {
    // Let's do 'cleanup' in suiteSetup too, in case another suite didn't clean up properly
    spies.restoreAll();
    stubs.restoreAll();
    console.log("I'm suiteSetup");
  },

  setup: function () {
    console.log("I'm setup");
    spies.create('log', console, 'log');
  },

  tearDown: function () {
    spies.restoreAll();
    console.log("I'm tearDown");
  },

  suiteTearDown: function () {
    console.log("I'm suiteTearDown");
    spies.restoreAll();
    stubs.restoreAll();
  },

  testSpies: function (test) {
    console.log('Hello world');
    expect(spies.log).to.have.been.calledWith('Hello world');
  },

  clientTestIsClient: function (test) {
    test.isTrue(Meteor.isClient);
    test.isFalse(Meteor.isServer);
  },

  serverTestIsServer: function(test){
    test.isTrue(Meteor.isServer);
    test.isFalse(Meteor.isClient);
  },

  tests: [
    {
      name: "skipped client test",
      type: 'client',
      skip: true,
      func: function (test) {
        test.isTrue(true)
      }
    },
    {
      name: "async test with timeout",
      timeout: 500,
      func: function (test, waitFor) {
        var onTimeout = function(){
          test.isTrue(true);
        };

        Meteor.setTimeout(waitFor(onTimeout), 50);
      }
    }
  ]
};

Munit.run(tddTestSuite);
```

## Sample Meteor App

Provided thanks to Michael Risse:

https://github.com/rissem/meteor-munit-example/

See the lib package munit tests there, including how to add your tests to your package.js:

https://github.com/rissem/meteor-munit-example/tree/master/packages/lib

## Running your package tests in the browser with hot code reloads

Assuming you develop your package as part of a meteor app and the package is located
in the packages folder, from the meteor app root, run:

`meteor test-packages package-name OR path-to-your-package [more packages]`

Then, just open your browser at the same url you use for your meteor app and the tests
will start running automatically, including re-run on every code change.

You can specify more than one package to test. Without arguments, it will test all packages in the packages folder, including the core meteor ones.

If you develop your package stand-alone, make sure meteor is in your path, and run:

`meteor test-packages path-to-your-package`

## Running your meteor app and your meteor package tests at the same time

The way we work internally is to run our meteor app with a free [mongohq](http://www.mongohq.com/)  sandbox database and at the same time run all of our packages tests with the internal meteor mongodb on a different port:

* app: `MONGO_URL=... meteor`
* tests: `unset MONGO_URL && export ROOT_URL=http://localhost:3100/ && meteor --port 3100 test-packages my-package1 my-package2`

For our convenience, we created a couple of shell scripts, one that runs the app and one that runs all our package tests, and that set / unset all the meteor related environment variables before hand. We recommend you do the same.

## Internals

The **Munit** test runner uses a slightly modified version of the `testAsyncMulti` function (with support for test timeouts) from the test-helpers package shipped with meteor to run all the tests in the test suite including all the setup and `tearDown` functions.

## Known Issues

* If a test fails, afterEach / tearDown will not be called. This is because MUnit uses testAsyncMulti behind the scenes, and this is a limitation of testAsyncMulti. We therefore recommend, as a workaround, to do cleanup in beforeEach / setup too.

* If the last test in a test suite fails, afterAll, suiteTearDown will not be called, for the same reason as above. We therefore recommend, as a workaround, to do cleanup in beforeAll / suiteSetup too.

## Contributions

Contributions are more than welcome. Here are some of our contributors:

* [@philcockfield](https://github.com/philcockfield) - added support for BDD style describe.it semantics.
* [@DominikGuzei](https://github.com/DominikGuzei) - added support for nested describe blocks.

## Changelog

[CHANGELOG](https://github.com/practicalmeteor/meteor-munit/blob/master/CHANGELOG.md)

## License

[MIT](https://github.com/practicalmeteor/meteor-munit/blob/master/LICENSE.txt)
