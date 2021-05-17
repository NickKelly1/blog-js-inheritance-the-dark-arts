# JavaScript Inheritance: the weird parts

## How well do you know JavaScript?

How many of these can you guess right?

### Class getters and setters

```javascript
class SuperClass {
  _value = undefined;
  get value() { return this._value; }
}
class SubClass extends SuperClass {
  set value(to) { this._value = to; }
}
const sub = new SubClass();
sub.value = 5;

// Q: what is logged?
console.log(sub.value); // undefined
```

### Deleting from an object

```javascript
const myObject = {
  deleteMe1: function() {},
  deleteMe2() {},
  toString() {},
};

delete myObject.deleteMe1;
// Q: what is logged?
console.log(myObject.deleteMe1); // undefined

delete myObject.deleteMe2;
// Q: what is logged?
console.log(myObject.deleteMe2); // undefined

delete myObject.toString;
// Q: what is logged?
console.log(myObject.toString); // toString() { [native code] }
```

### Deleting from a class

```javascript
class MyClass {
  deleteMe1 = function() {}
  deleteMe2() {}
}
const myInstance = new MyClass();

delete myInstance.deleteMe1;
// Q: what is logged?
console.log(myInstance.deleteMe1); // undefined

delete myInstance.deleteMe2;
// Q: what is logged?
console.log(myInstance.deleteMe2); // deleteMe2() {}
```

If you got them all right? If so maybe you're already battle-worn JavaScript veteran, but otherwise let me save you some trouble.

 already spent countless hours fighting JavaScripts prototypal inheritance. If not, let me save your future self some time!

## Inheritance

In object-oriented programming, inheritance is the mechanism used to base an object or class on a another parent object or class.

JavaScripts has inheritance but doesn't have "classes" like most other OOP languages do. Instead, JavaScript links objects together by "[prototypes](https://en.wikipedia.org/wiki/Prototype-based_programming)" that convery inheritance. Even ES2015 JavaScript [`classes`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) are mostly syntactic sugar for objects with prototypal relationships.

In this post we'll discuss the deep, dark secrets of JavaScripts prototypal inheritance: the flexibility, the gotchas, and the just-plain-weird bits.

## The basics

To get started we need to talk about how we use inheritance day-to-day.

Here's feature we might expect from inheritance in a sensible language:

1. `base` is an object
2. `base` has a property `baseProp`, accessible via `base.baseProp`
3. `sub` is an object that inherits from base
4. Therefore, `sub` inherits property `baseProp`, accessible via `sub.baseProp`

That is,

```javascript
class Base {
  baseProp = 'hello world';
}
class Sub extends Base {
  //
}
const sub = new Sub();
// sub has access to properties on base
console.log(sub.baseProp);  // "hello world"
```

So-far everything appears sane 👌

## Prototypes

JavaScript uses `prototypes` to achieve inheritance. All objects have a `[[Prototype]]` [internal slot](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots) which is the object being inherited from. Internal slots are values internal to the JavaScript interpreter that may or may not be exposed to JavaScript code.

An objects `[[Prototype]]`s can be null or another object which itself has a `[[Prototye]]` slot. An objects linked list of `[[Prototype]]`'s is called its "prototype chain" and terminates with null.

We differentiate between the `[[Prototype]]` internal slot of objects (used for inheritance), and the `prototype` property of non-arrow function objects which we'll see soon.

We can use [`Object.getPrototypeOf(obj)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf) to get the `[[Prototype]]` of an object `obj`.

```javascript
// For the prototype chain: child (inherits) parent (inherits) ancestor:
Object.getPrototypeOf(child) === parent;    // child inherits from parent
Object.getPrototypeOf(parent) === ancestor; // parent inherits from ancestor
Object.getPrototypeOf(ancestor) === null;   // ancestor inherits nothing
```

We can also use [`Object.setPrototypeOf(sub, base)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf) to change the `[[Prototype]]` of an object `sub` to another object (or `null`), `base`. Notice - unlike static OO languages we can dynamically change inheritance heirarchies at runtime! For performance reasons this is *strongly* advised against. According to Benedikt Muerer of v8, [a every time you change the prototype chain, a kitten dies](https://youtu.be/IFWulQnM5E0?t=1669).

```javascript
const base = { prop: 'hello world' };
const sub = {};
sub.base; // undefined
Object.setPrototyeOf(sub, base);
sub.base; // "hello world"
Object.setPrototyeOf(sub, null);
sub.base; // undefined
```

Objects created using the object literal syntax `{}` inherit from JavaScript's base `Object.prototype`. `Object.prototype` inherits from `null`.

```js
const obj = {};
Object.getPrototypeOf(obj) === Object.prototype; // true
Object.getPrototypeOf(Object.prototype) === null; // true
```

## Constructors

Constructors are the other half of prototypes.

Any non-arrow function (any function created with the `function` keyword) can be a constructor. Because of this it's good practice to begin function names intended to be used as constructors with an uppercase character so the consumer is aware.

Constructors are not strictly necessary to use inheritance in JavaScript but they provide a consistent way to create prototypes and construct objects inheriting them.

It's important to understand that functions also objects but with a `[[Call]]` internal slot. Like other objects, functions have their own `[[Prototype]]` internal slot.

All non-arrow functions `Fn` are given a `prototype` property `Fn.prototype` initialised to a new object `{}`. Don't get `Fn.prototype` confused with `Fn's` `[[Prototype]]` internal slot which is what `Fn` inherits from. `Fn.prototype` will become the `[[Prototype]]` of objects constructed from `Fn` when called with the `new` keyword `const obj = new Fn()`.

Constructors need to be called with the `new` keyword to work as intended. Constructors called with `new` have access to a special `this` object in their body; a freshly created object whose ``[[Prototype]]`` is the constructor functions `prototype` property. Constructors implicitly return their `this` object.

As a side note, classes created with the ES2015 `class` (`class MyClass {...}`) are also simply constructor functions, but have an internal slot [`[[IsClassConstructor]]`](https://tc39.es/ecma262/#sec-ecmascript-function-objects) that [causes them to throw](https://tc39.es/ecma262/#sec-ecmascript-function-objects-call-thisargument-argumentslist) a `TypeError` if called without the `new` operator unlike functions not created with the `class` syntax.

```javascript
// By convention we capitalise constructors so it's clear they are not normal functions and should be called with the `new` keyword
function MyConstructor() {
  // thanks to the `new operator`, a new object has been created and is
  // accessible through the `this` variable in the constructors body

  // The `[[Prototype]]` of `this` is the prototype property of `MyConstructor`
  Object.getPrototypeOf(this) === MyConstructor.prototype; // true
  this.value; // 'hello world'

  // `this` is implicitly returned
}

MyConstructor.prototype.value = 'hello world';

const myInstance = new MyConstructor();
myInstance.value; // 'hello world'
```

Given that instances created with the `new` operator inherit from (have their `[[Prototype]]` set to) their constructors `Fn.prototype`, we can create something that looks like proper classes.

```javascript
function Person() {
  //
}

Person.prototype.sayHello = function() {
  console.log('hello');
}

const person = new Person();
person.sayHello();  // 'hello'
```

## Simulating ES2015 classes with es5

Knowing about prototypes and constructors we can create so-called classes inherit from other so-called classes.

Using es5 constructor-prototype syntax (i.e. not ES2015 `class` syntax), we have enormous flexibility in how we string together our objects but we have to string them together manually. We can manually accomplish what the ES2015 `class` syntax does for us by ensuring:

- **Instance prototype chain**: `SubClass.prototype.[[Prototype]]` must be set to `SuperClass.prototype`. This sets up the prototype chain of instances constructed from `new SubClass(...)` such that:
  - `subclass_instance.[[Prototype]]` === SubClass.prototype
  - `subclass_instance.[[Prototype]][[Prototype]]` === SuperClass.prototype
  - `subclass_instance.[[Prototype]][[Prototype]][[Prototype]]` === Object.prototype
  - `subclass_instance.[[Prototype]][[Prototype]][[Prototype]][[Prototype]]` === null
- **Constructor prototype chain**: `SubClass.[[Prototype]]` must be set to `SuperClass`. This means the `SubClass` function inherits "static" properties from `SuperClass` (properties on the SuperClass constructor function) such that:
  - `SuperClass.staticProperty = 5`
  - `SubClass.staticProperty === 5`
- **Initialisation**: When the `SubClass` constructor is called with `new`, it needs to immediately call the `SuperClass` constructor function binding its `this` value (`SuperClass.call(this, ...)`), in order to initialise the `SuperClass` on `this` properly.
  - The ES2015 `class` syntax forces us to call the super constructor using `super()` at the beginning of our subclasses constructor function, or else the interpreter will throw an error. This is not forced in es5 constructor functions so we need to remember it ourselves! Otherwise our class map not be properly initialised.

## Es5 classes by example

We'll code a simple inheritance example of 2 classes in an ORM, a base `Model` superclass and `User` subclass. Below is a diagram representing the various ways our objects will be connected together. Each inheritance layer has 3 associated objects: the constructor function, prototype object and instance object.

Don't be intimidated by the amount of objects and connections in a simple 1-level inheritance problem - if you can grok the diagram and code below then you can derive an understanding of everything relating to JavaScript prototypes, inheritance and OOP.

```txt
- 2 "classes": `Model` and `User`
- `User` inherits from `Model`

    |================================|
    |         ModelConstructor      <-------------------------|--|--|
    |================================|                        |  |  |
    |-properties:--------------------|                        |  |  |
    |   abstract table: string       |                        |  |  |
    |   prototype: ModelPrototype   >---|                     |  |  |
    |-methods:-----------------------|  |                     |  |  |
    |   createQueryBuilder: unknown  |  |                     |  |  |
    |================================|  |                     |  |  |
                                        |                     |  |  |
                     |------------------|                     |  |  |
                     |                                        |  |  |
                     |  |==================================|  |  |  |
|-----------------|--|--->        ModelPrototype           |  |  |  |
|                 |     |==================================|  |  |  |
|                 |     |-methods:-------------------------|  |  |  |
|                 |     |   toDTO(): object                |  |  |  |
|                 |     |-properties:----------------------|  |  |  |
|                 |     |   readonly table: string         |  |  |  |
|                 |     |-magic properties:----------------|  |  |  |
|                 |     |   constructor: ModelConstructor >---|  |  |
|                 |     |==================================|     |  |
|                 |                                              |  |
|                 |----------------------------|                 |  |
|                                              |                 |  |
|      |====================================|  |                 |  |
|  |---->         ModelInstance             |  |                 |  |
|  |   |====================================|  |                 |  |
|  |   |-magic properties-------------------|  |                 |  |
|  |   |   constructor                     >-------------------- |  |
|  |   |-internal slots:--------------------|  |                    |
|  |   |   [[Prototype]]: ModelConstructor >---|                    |
|  |   |====================================|                       |
|  |                                                                |
|  |                                                                |
|  |  |=========================================================|   |
|  |  |                     UserConstructor                     |   |
|  |  |=========================================================|   |
|  |  |-methods:------------------------------------------------|   |
|  |  |   toDTO(): object                                       |   |
|  |  |-properties:---------------------------------------------|   |
|  |  |   readonly table: string                                |   |
|  |  |   prototype: UserPrototype                              |   |
|  |  |-internal slots:-----------------------------------------|   |
|  |  |  [[Prototype]]: ModelConstructor                       >----|
|  |  |  [[Construct]](id: string, name: string): UserInstance >---|
|  |  |=========================================================|  |
|  |                                                               |
|  |  |------------------------------------------------------------|
|  |  |
|--|--|---------------------------------------------|
   |  |                                             |
   |  |     |==================================|    |
   |  |  |--->          UserPrototype          |    |
   |  |  |  |==================================|    |
   |  |  |  |-internal slots:------------------|    |
   |  |  |  |  [[Prototype]]: ModelPrototype  >-----|
   |  |  |  |==================================|
   |  |  |                                      
   |  |  |-----------------------------------------|
   |  |                                            |
   |--|-----------------------------------------|  |
      |                                         |  |
      |  |===================================|  |  |
      |--->          UserInstance            |  |  |
         |===================================|  |  |
         |-internal slots:-------------------|  |  |
         |  [[Prototype]]: UserPrototype    >------|
         |-builds from:----------------------|  |
         |  ModelInstance                   >---|
         |===================================|
```

```javascript
// ******************
// @Filename model.js
// ******************

// Model constructor
// -----------------

/**
 * @constructor
 * @abstract
 * @classdesc
 * ORM SuperClass representing a persisted model
 *
 * @param {undefined|string|number} id
 * @returns {Model}
 */
function Model(id) {
  /**
   * @public
   * @property {number|string|undefined} id
   */
  this.id = id;
}

// static properties and methods
// -----------------------------

/**
 * @public
 * @static
 * @abstract
 * @property {string} table
 *
 * Database table name of the model
 * To be overridden by subclasses
 */
Model.table = undefined;

/**
 * Create a new model by cloning an old one
 * (can be overridden on subclasses constructor)
 *
 * @static
 * @param {this} from
 * @returns {this['prototype']}
 */
Model.createQueryBuilder = function(from) {
  // create a query builder using the static, overridden table name
  // as the target table
  return db.createQueryBuilder(this.table);
}


// prototype methods and properties
// -------------------------------------------

/**
 * @property {string} Model.prototype.table
 * 
 * Get the table name
 *
 * (Create a getter on the superclass Model prototype to get the tables name
 * from the overridden static property `table` on its subclasses)
 */
Object.defineProperty(Model.prototype, 'table', {
  /**
   * Get the table name from the model instance
   * @returns {string}
   */
  get() {
    // expect super.constructor.table to be a static,
    // overridden property
    return Object.getPrototypeOf(this).constructor.table;
  },
});


/**
 * Deserialize the model to a DTO (Data Transfer Object)
 */
Model.prototype.ToDTO = function() {
  return { id: this.id };
}

// ...

// *****************
// @Filename user.js
// *****************

// User constructor
// ----------------

/**
 * @constructor
 * @extends Model
 * @classdesc
 * SubClass ORM model representing a persisted user
 *
 * @param {string} id
 * @param {string} name
 * @returns {User}
 */
function User(id, name) {
  // immediately call the SuperClasses constructor to initialise
  // the superclass
  // (alterantively: `Model.call(this, id)`)
  Object
    .getPrototypeOf(Object.getPrototypeOf(this))
    .constructor
    .call(this, id);


  /** @public @override @property {string} id */
  this.id = id;
  /** @public @property {string} name */
  this.name = name;
}

// New instances of User should inherit from Model
// Prototype chain:
//  `userInstance` -> `User.prototype` -> `Model.prototype` -> ...
Object.setPrototypeOf(User.prototype, Model.prototype);

// The User constructor function itself should extend from the Model constructor function to inherit static methods
// Prototype chain:
// `User` -> `Model` -> `Function.prototype` -> ...
Object.setPrototypeOf(User, Model);

// Static properties and methods
// -----------------------------

/** @override */
User.table = 'users';

// prototype (instance) methods and properties
// -------------------------------------------

/** @override */
User.prototype.toDTO = function() {
  return {
    // super call - equivalent to `super.toDTO()`
    ...Object.getPrototypeOf(this).toDTO(),
    name: this.name
  };
}

// ****************
// @Filename app.js
// ****************

const userInstance = new User('da935633-1c49-4d88-aaeb-79f244e8cb83', 'Nick');
```

## Class inheritance vs Object inheritance

In JavaScript, classes are simply objects with some distinct properties. 

Here's where things begin to fly off the rales for most developers familiar with class-based object oriented languages.

In JavaScript, objects can inherit from other objects. As we

```javascript
```
