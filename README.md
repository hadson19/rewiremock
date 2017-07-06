# rewiremock [![Build Status](https://secure.travis-ci.org/theKashey/rewiremock.svg)](http://travis-ci.org/theKashey/rewiremock)

[![NPM](https://nodei.co/npm/rewiremock.png?downloads=true&stars=true)](https://nodei.co/npm/rewiremock/)

Simple es6-friendly mocking library inspired by the best libraries:
- [mockery](https://github.com/mfncooper/mockery) - the library I like more.
- [mock-require](https://github.com/boblauer/mock-require) - the simplest thing ever.
- [proxyquire](https://github.com/theKashey/proxyquire) - the one I know better than others.

By its nature rewiremock has same behavior as Mockery. But it can have any behavior.
It covers _any_ cases.

Rewiremock is an evolution of my way of explorations: 
the better [proxyquire](https://github.com/theKashey/proxyquire), 
the way of[resolveQuire](https://github.com/theKashey/resolveQuire),
and magic of [proxyquire-webpack-alias](https://github.com/theKashey/proxyquire-webpack-alias).

Rewiremock was initially named as mockImports or mockModule. But was renamed for WireMock.

We shall not use that name, but [rewire](https://github.com/jhnns/rewire) - is one of existing micking library.


# Idealogy
- be simply
- be modular
- be secure
- be fast

# API
 see d.ts file, or JSDoc in javascript sources.
 
 ## main API
 - rewiremock.enable() - wipes cache and enables interceptor.
 - rewiremock.disable() - wipes cache and disables interceptor.
 - rewuremock.inScope(loader) - loads a module in sandbox.
 ## mocking API 
 - rewiremock(moduleName: string):rewiremock - set name of overloading module
    - .with(stubs: function | Object) - overloads current module
    - .withDefault(stub: function | Object) - overload `default` es6 export
    - .by(otherModule: string) - overload everything by another module
    - .callThought() - first load original module, and next extend it by provided stub.
    - .enable/disable() - to enable or disable mock (enabled by default).
    - .toBeUsed() - enables   
 ## isolation API
 - rewiremock.isolation() - enables isolation
 - rewiremock.withoutIsolation() - enables isolation
 - rewiremock.passBy(pattern or function) - enables some modules to pass thought isolation.

# Setup

First - define your mocks. You can do it in any place, this is just a setup.
```javascript
 import rewiremock from 'rewiremock';
 ...
 
 // totaly mock `fs` with your stub 
 rewiremock('fs')
    .with({
        readFile: yourFunction
    });
  
 // replace path, by other module 
 rewiremock('path')
    .by('path-mock');
  
 // replace default export of ES6 module 
 rewiremock('reactComponent')
    .withDefault(MockedComponent)
 
 // replace only part of some library and keep the rest 
 rewiremock('someLibrary')
    .callThought() 
    .with({
        onlyOneMethod
    })
```   
   
# Running
 There is a simply way to do it: Just enable it, and dont forget to disable.
 ```javascript
  //in mocha tests
  beforeEach( () => rewiremock.enable() );
  //...
  // here you will get some advantage in type casting and autocompleting.
  // it will actually works...
  const someModule = require('someModule'); 
  //...
  afterEach( () => rewiremock.disable() );
 ```
 On enable rewiremock will wipe from cache all mocked modules, and all modules which requires them.
 
 Including your test.
 
 On disable it will repeat operation. 
 
 All test unrelated modules will be keept. Node modules, react, common files - everything.
 
 As result - it will run faster.
 
# inScope
 And there is a bit harder way to do it - scope.
 inScope will create new internal scope, next you can add something new to it, and then it will be destroyed.
 It will also enable/disable rewiremock just in time.
 
 This helps keep tests in isolation.
 
 PS: scopes are nesting each other like javascript prototypes do.
```javascript
rewiremock.inScope(
    () => import('somemodule'), // load a module. Using import or require.
    // just before it you can specify mocks or anything else
    (mock) => { 
        addPlugin(nodePlugin);

        mock('./lib/a/foo').with(() => 'aa');
        mock('./lib/a/../b/bar').with(() => 'bb');
        mock('./lib/a/../b/baz').with(() => 'cc');
    }
) // at this point scope is dead
    .then((mockedBaz) => { 
        expect(mockedBaz()).to.be.equal('aabbcc');
    });
```  
or just 
```javascript
rewiremock.inScope(() => import('somemodule')).then(mockedModule => doSomething)  
```
or
```javascript
rewiremock.inScope(
    () => import('somemodule').then( mockedModule => doSomething),    
    (mock) => aPromise   
);

```
Currently .inScope is the only API capable to handle es6 dynamic imports.

# Plugins
 By default - rewiremock has limited features. You can extend it behavior by using plugins.
 - relative. A bit sily, proxyquire-like behavior. Will override only first level deps, and will wipe a lot of modules from a cache.
 - nodejs. Common support to `usual` node.js application. Will absolutize all paths. Will wipe cache very accurately. 
 - webpack-alias. Enabled you to use webpack aliases as module names.
 - childOnly. Only first level dependencies will be mocked.
 - disabledByDefault. All mocks will be disabled on create and at the end of each cycle.
 - protectNodeModules. Ensures that any module from node_modules will not be wiped from a cache.  
 ```javascript
 import rewiremock, { addPlugin, removePlugin, plugins } from 'rewiremock';     
 
 addPlugin(plugins.webpackAlias);
 removePlugin(plugins.webpackAlias);
 ``` 

# Nested declarations
 If you import rewiremock from other place, for example to add some defaults mocks - it will not gonna work.
 Each instance of rewiremock in independent.
 You have to pass your instance of rewiremock to build a library.
 PS: note, rewiremock did have nested API, but it were removed.
  
# Isolation
 Unit testing requires all decencies to be mocked. All!
 To enable it, run
 ```javascript
  rewiremock.isolation();
  //or
  rewiremock.withoutIsolation();
 ```
 Then active - rewiremock will throw error on require of any unknown module.
 
 Unknown is module which is nor mocked, nor marked as passthrough. 
 
 To enable few modules to in `invisible` to rewiremock run
 ```javascript
 rewiremock.passBy(/*pattern or function*/);
 
 rewiremock.passBy(/common/);
 rewiremock.passBy(/React/);
 rewiremock.passBy(/node_modules/);
 rewiremock.passBy((name) => name.indexOf('.node')>=0 )
 ```
 
 # Reverse isolation.
  Sometimes you have to be sure, that you mock is actually was called.
  Isolation will protect you then you add new dependencies, .toBeUsed protect you from removal.
 
 # Your own setup.
  In most cases you have to:
   - add plugin
   - setup default passBy rules
   - add some common mocks
   - do something else.
   
   And it is not a good idea to do it in every test you have.
   
   It is better to have one setup file, which will do everything for you
   * Part one - man in the middle
```javascript
  // this is your test file
  
  // instead of importing original file - import your own one
  // import rewiremock from 'rewiremock';
  import rewiremock from 'test/rewiremock';    
```
  
  * Part 2 - create your own one
    
```javascript
    // this tests/rewiremock.js
    
    import rewiremock, { addPlugin, overrideEntryPoint} from 'rewiremock';
    // do anything you need
    addPlugin(something);
    rewiremock('somemodule').with(/*....*/);   
    
    // but dont forget to add some magic
    overrideEntryPoint(module); // <-- set yourself as top module
    // PS: rewiremock will wipe this module from cache to keep magics alive.
       
    export default rewiremock;
``` 
 * Part 3 - enjoy.
 You extract some common code into helper. A lot of things become easer.
   
 # Caching
 
Other libraries will always do a strange things with a cache:

- just wipe everything. Absolutely. 
As result tests will run slowly, or even did not run at all. 
Normaly you should not wipe native(.node) modules, and external(node_modules) modules.
 For example you should not wipe React - _new_ version of React will be incompatible with old one.
- wipe only listed modules. Exactly.
 Also not a good idea, as long sometimes between first module, and mocked one you can found some sort of middleware.
 Syntax sugar, thirdparty library, helper, and so on.

Rewiremock uses a bit different, more smart way:

- all files required from original test, while interceptor is active, will bypass cache.
 (proxyquire cant do it, as long it work at more low level API).
- all files you indicate as mock will be removed from cache. Unfortunately all, no matter of usage.
- all files which use any wiped ones - will be also removed from a cache.
- repeat.
  
As result - it will never wipe something it should not.

As result - you can mock any file at any level. Sometimes it is usefull.
  
If you __dont want__  this - just add `relative` plugin. It will allow mocking only for modules

>  _required from __module__ with __parent__ equals __entryPoint__._

  PS: module with parent equals entryPoint - any module you require from test(it is an entry point).
  required from that module - first level require. Simple.
  
  
 # Own plugins
Dont forget - you can write your own plugins. 
 plugin is an object with fields:
 ```javascript
 {
 // to transform name. Used by alias or node.js module
 fileNameTransformer: (fileName, parentModule) => fileName;
 // check should you wipe module or not. Never used :)
 wipeCheck: (stubs, moduleName) => boolean,
 // check is mocking allowed for a module. User in relative plugin
 shouldMock: (mock, requestFilename, parentModule, entryPoint) => boolean
 }
 ```
 
# Licence
 MIT
 
 

Happy mocking!
