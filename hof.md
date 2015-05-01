Introduction to higher-order functions
======================================

Having the possibility to use functions as values in a programming language enables us to make [higher-order functions][hof]. That's a fancy way of saying that **a function receives and/or returns functions**. Lambdas, Closures, Blocks and Methods are also fancy names to say: *functions* (That's an oversimplification but will be enough for this post).
That may seem simple but has tremendous implications. We are going to explore some of them here. 

[hof]: http://en.wikipedia.org/wiki/Higher-order_function

To get better understanding of the mechanics, we are going to reimplement some [higher-order functions][hof] and that will give us an intuition of what may happen under the covers.

Starting point
--------------

```js
var minimumAgeToDrive = 17;
var people = [
  {name: "John", age: 17},
  {name: "Jane", age: 15},
];

for (var i = 0; i < people.length; i++) { 
  if (people[i].age >= minimumAgeToDrive) {
    console.log(people[i].name);
  }
}
```

This `for` loop, although small and harmless looking, contains highly entangled code. It can only be tested as a whole. It is full of concreteness. Nothing in it can be reused. Users are going to be forced to duplicate the loop and most of the operations if they want to do something similar.

For example: let's suppose in some parts we want to print the names of the people who can drive. And, in other parts we want to save them in a file. We want to reuse everything but the process of doing something with the names of each eligible driver. What do we do?

```js
var eligibleDriverNames = [];
for (var i = 0; i < people.length; i++) { 
  if (people[i].age >= minimumAgeToDrive) {
    eligibleDriverNames.push(people[i].name);
  }
}

//in some part of the code:
for (var i = 0; i < eligibleDriverNames.length; i++) { 
  console.log(eligibleDriverNames[i]);
}

//in other part of the code:
for (var i = 0; i < eligibleDriverNames.length; i++) { 
  fs.appendFileSync('file', eligibleDriverNames[i]);
}
```

We have here structural duplication. We are repeating the same loop structure each time we do something with each element of a collection. The only difference between the last two loops is the place where we write the `name`. Because the code in those have completelly different APIs we need to introduce a common public interface to treat them as *similar things* in the sense that they can `write` a `text`. In an object oriented language we could do that like this:

```js
var consoleStream = {
  write: function(text) {
    console.log(text);
  }
};
var fileStream = {
  write: function(text) {
    fs.appendFileSync('file', text);
  }
};

var writeEligibleDriverNames = function(stream) {
  for (var i = 0; i < eligibleDriverNames.length; i++) { 
    stream.write(eligibleDriverNames[i]);
  }
};

//in some part of the code:
writeEligibleDriverNames(consoleStream);

//in other part of the code:
writeEligibleDriverNames(fileStream);
```

But that will solve a part of the problem and only for this concrete scenario. We don't want to be tied to `eligibleDriverNames` so we can reuse it in more scenarios. We could generalise it a bit more by passing it as a parameter like this:

```js
var writeEach = function(collection, stream) {
  for (var i = 0; i < collection.length; i++) { 
    stream.write(collection[i]);
  }
};

//in some part of the code:
writeEach(eligibleDriverNames, consoleStream);

//in other part of the code:
writeEach(eligibleDriverNames, fileStream);
```

That code now is more generic but it is completely tied to the `write` method on the `stream` object. We want a more generic solution, one that we can reuse in a wider context. We need to be able to do anything with each element of the collection, not only `write` on a `stream`.

Receive functions as arguments
------------------------------

To swap the code that we execute on `each` element of a collection we previously introduced the `stream` *duck type*. That worked because we introduced a public API that was common to the options we had at the moment. We introduced a bit of polymorphism. If we want to make it more generic, we need to remove more specific details from that API. You can think that someone receives a `text` and the outcome of that call is something that cannot be seen by the caller. That way we have removed the `stream` and `write` concepts. What we have left is just a function that receives a `text` and does not return anything. That results in much less restricted polymorphism.

```js
var each = function(collection, sideEffect) {
  for (var i = 0; i < collection.length; i++) { 
    sideEffect(collection[i]);
  }
};

//in some part of the code:
each(eligibleDriverNames, consoleStream.write);

//in other part of the code:
each(eligibleDriverNames, fileStream.write);
```

Now, this is `each` function is very generic. It can be reused in a very wide context.

We can do the same with the remaining piece of code from the beginning:

```js
var eligibleDriverNames = [];
for (var i = 0; i < people.length; i++) { 
  if (people[i].age >= minimumAgeToDrive) {
    eligibleDriverNames.push(people[i].name);
  }
}
```

To be able to do that we need to separate the parts involved in [each responsibility][srp]. Let's generalise the one that `filters` the drivers first:

[srp]: http://en.wikipedia.org/wiki/Single_responsibility_principle

```js
var drivers = [];
for (var i = 0; i < people.length; i++) { 
  if (people[i].age >= minimumAgeToDrive) {
    eligibleDriverNames.push(people[i]);
  }
}
```

```js
var filter = function(collection, predicate) {
  var filtered = [];
  for (var i = 0; i < collection.length; i++) { 
    if (predicate(collection[i])) {
      filtered.push(collection[i]);
    }
  }
  return filtered;
};

var canDrive = function(person){
  return person.age >= minimumAgeToDrive;
};
var drivers = filter(people, canDrive);
```

And now the one that `maps` each driver to his/her name:

```js
var eligibleDriverNames = [];
for (var i = 0; i < drivers.length; i++) { 
  eligibleDriverNames.push(drivers[i].name);
}
```

```js
var map = function(collection, transformation){
  var mapped = [];
  for (var i = 0; i < collection.length; i++) { 
    mapped.push(transformation(collection[i]));
  }
  return mapped;
}

var getName = function(person){
  return person.name; 
};
var eligibleDriverNames = map(drivers, getName);
```

By hiding the `for` loop inside those functions, we have created an [abstraction][abstraction]. We have separated *what to do* from *how to do it*. Now the clients can depend only on *what to do*. That enables the possibility to change *how to do it* without affecting them.

[abstraction]: http://en.wikipedia.org/wiki/Abstraction_(computer_science)

For example: In the `each` function *what we do* is: *do a `sideEffect` with each element in the collection*. *How we do it* is: *iterating sequentially with the `for` loop*.

We could change the implementation of `each` for one that process the elements in parallel to take advantage of multi-core or several clusters. We could also change the implementation of `map` for one that does the transformation only if the result is going to be used ([lazy evaluation][lazy]).

[lazy]: http://en.wikipedia.org/wiki/Lazy_evaluation

Return functions
----------------

Let's imagine we want to `filter` the `people` in different ways. Not only the ones who `canDrive`. Let's suppose we also want to `filter` the `people` who are named "Jane".

```js
var drivers = filter(people, canDrive);

var isNamedJane = function(person){
  return person.name == "Jane";
};
var janes = filter(people, isNamedJane);
```

We are passing the `people` to each one of those calls. If we were only filtering `people`, that may seem redundant. We could define a function that only filters on the `people`.

```js
var filterPeople = function(predicate){
  return filter(people, predicate);
};

var drivers = filterPeople(canDrive);
var janes = filterPeople(isNamedJane);
```

But that only works in this context because it is tied directly to `people`. We would like to do that with any collection.

```js
var filterCollection = function(collection){
  return function(predicate){
    return filter(collection, predicate);
  };
};
var filterPeople = filterCollection(people);

var drivers = filterPeople(canDrive);
var janes = filterPeople(isNamedJane);
```

The result of the function `filterCollection` is another function with one parameter less. That is called [partial application][partial]. Which means that we are predefining arguments.

[partial]: http://en.wikipedia.org/wiki/Partial_application

But that is only valid for the `filter` function. We would like to do that with any function and any amount of arguments.

```js
var bind = function(initialFunction, argument1, argument2, argumentN){
  var partialArguments = Array.prototype.slice.call(arguments, 1);
  return function(){
    var remainingArguments = Array.prototype.slice.call(arguments);
    return initialFunction.apply(null, partialArguments.concat(remainingArguments));
  };
};
var filterPeople = bind(filter, people);

var drivers = filterPeople(canDrive);
var janes = filterPeople(isNamedJane);
```

Now we can only `bind` the first arguments to a function. We would like to `bind` any combination of arguments to a function. But this time, intead of trying to generalise `bind` for that purpose, we are going to `cycle` the arguments of a function so we can then `bind` the ones we prefer.

```js
var cycle = function(initialFunction) {
  return function(argument1, argument2, argumentN){
    var args = Array.prototype.slice.call(arguments);
    args.push(args.shift());
    return initialFunction.apply(null, args);
  };
};
var filterDrivers = bind(cycle(filter), canDrive);
var filterJanes = bind(cycle(filter), isNamedJane);

var drivers = filterDrivers(people);
var janes = filterJanes(people);
```

The last thing we would like to do is to `join` several functions into one, having the output from one function going to the input of the next. With that, we can reproduce the behaviour of the first piece of code in the starting point.

```js
var join = function(function1, function2, functionN) {
  var functions = arguments;
  return function(input) {
    var output = input;
    for (var i = 0; i < functions.length; i++) {
      output = functions[i](output);
    }
    return output;
  };
};

var onlyDrivers = bind(cycle(filter), canDrive);
var getTheirNames = bind(cycle(map), getName);
var printThem = bind(cycle(each), consoleStream.write);

join(onlyDrivers, getTheirNames, printThem)(people);
```

There we have code at a much higher level of abstraction. In contrast with the code at the starting point, this one enables us to see what the program is doing in human terms: given `people` treats `only drivers`, `gets their names` and `prints them`.

The functions that we've defined conform to SOLID principles at a much more fine-grained level. That makes them very loosely coupled and as a result they are extremely easy to test as units. In addition, if we need to change a part of the system, it is as simple as changing a function.

You may start to feel now that [higher-order functions][hof] have very nice composable properties. I like to see them as a way to create a kind of lego system, a system with very composable little pieces that makes easy to create big and very expressive worlds.

![lego](http://cdn.collider.com/wp-content/uploads/lego-rivendell-lord-of-the-rings-2.jpg)
