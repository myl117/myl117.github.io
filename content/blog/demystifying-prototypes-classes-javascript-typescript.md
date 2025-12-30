+++
title = "Demystifying Prototypes and Classes in JavaScript (and TypeScript)"
date = "2025-12-30"
tags = [
    "javascript",
    "typescript",
]
+++

If you’ve worked within the JavaScript ecosystem, you’ve probably used an ES6 class data structure. But you may not know about its more primitive counterpart: prototypes. The core truth is that JavaScript is prototype-based language and one of the only mainstream object-oriented languages to use prototypal inheritance.

Classes are just syntax sugar over prototypes and throwing TypeScript into the mix adds type safety but does not enforce runtime behaviour.

But what are prototypes? A prototype is just another object that an object delegates to when a property or method is not found on itself and this allows for shared methods across multiple instances of an object. Every object gets a _[[Prototype]]_ slot automatically when it is created.

Let’s dive into some practical examples of objects with and without using a prototype-based method. Here is an example with a method directly on an object (no prototype):

```ts
function Animal(this: any, type: string) {
  this.type = type;
  this.sayHi = function () {
    console.log(`Hi, I'm a ${type}`);
  };
}

const cat = new (Animal as any)("Cat");
const dog = new (Animal as any)("Dog");

cat.sayHi();
dog.sayHi();
```

This works fine but every instance of our animal gets its own copy of the _sayHi_ method. Imagine if we had 100,000 animals in our application. That would not be good as memory usage would grow linearly.

Now let's look at a prototype method:

```ts
function Animal(this: any, type: string) {
  this.type = type;
}

Animal.prototype.sayHi = function () {
  console.log(`Hi, I'm a ${this.type}`);
};

const cat = new (Animal as any)("Cat");
const dog = new (Animal as any)("Dog");

cat.sayHi();
dog.sayHi();
```

We can confirm that the _sayHi_ method is now shared in memory by checking if both the cat and dog methods are the same.

```ts
console.log(cat.sayHi === dog.sayHi);
```

Okay, so now we know the gist of how prototypes work, how can we extend this prototype constructor so that one object inherits from another? Let's create a Dog object that inherits from our Animal object.

```ts
function Bird(this: any, type: string, species: string) {
  Animal.call(this, type);
  this.species = species;
}

// Set up prototype chain
Bird.prototype = Object.create(Animal.prototype);

// Restore constructor reference
Bird.prototype.constructor = Bird;

// Add a new method specifically for our Bird object
Animal.prototype.sing = function () {
  console.log(`*${this.species} singing sounds*`);
};

const canary = new (Bird as any)("Bird", "Canary");

canary.sayHi(); // Hi, I'm a Canary (inherited from Animal)
canary.sing(); // *Canary singing sounds* (own method)
```

We can confirm that our canary is an instance of our Bird object which itself is an instance of our Animal object

```ts
console.log(canary instanceof Bird);
console.log(canary instanceof Animal);
```

![Prototype chain diagram](/images/demystifying-prototypes-classes-javascript-typescript/diagram.jpg)
_Prototype chain diagram (Guess I'm a graphic designer now ;)_

Okay so now that we understand the fundamentals of prototypes, translating them into ES6 classes is straightforward especially if you are familiar with OOP principles already. Let's translate our Animal and Bird objects into ES6 classes.

```ts
class Animal {
  type: string;

  constructor(type: string) {
    this.type = type;
  }

  sayHi() {
    console.log(`Hi, I'm a ${this.type}`);
  }
}

class Bird extends Animal {
  species: string;

  constructor(type: string, species: string) {
    super(type);
    this.species = species;
  }

  sing() {
    console.log(`*${this.species} singing sounds*`);
  }
}

const canary = new (Bird as any)("Bird", "Canary");

canary.sayHi();
canary.sing();
```

So to recap, we've learnt about how prototypes and constructors worked in ES5. How ES6 introduced classes which are syntax sugar over prototypes. Let's now talk about type safety. Typescript doesn't change the underlying runtime behaviour of JavaScript but instead does add types and modifiers for compile-time safety:

1. public - can be used anywhere
2. private - only in class
3. protected - inside class and subclasses

```ts
class Person {
  public name = "Alice";
  protected age = 25;
  private secret = "it's a secret ;)";
  readonly id = 1;
}

class Employee extends Person {
  showInfo() {
    console.log(this.name); // public ✅
    console.log(this.age); // protected ✅
    // console.log(this.secret); ❌ private, not accessible
    console.log(this.id); // readonly ✅
  }
}

const e = new Employee();
console.log(e.name); // ✅ public
// console.log(e.age);    ❌ protected
// console.log(e.secret); ❌ private
// e.id = 2; ❌ readonly
e.showInfo();
```

And that’s the fundamentals of prototypes and classes in JS/TS. I hope this has cleared up how they relate and made the underlying model feel a lot less “black-boxed” and more intelligible.

Thanks for reading :)
