Introduction to higher-order functions
======================================

Having the possibility to use functions as values in a programming language enables us to make [higher-order functions][hof]. That's a fancy way of saying that **a function receives and/or returns functions**. Lambdas, Closures, Blocks and Methods are also fancy names to say: *functions* (That's an oversimplification but will be enough for this post).

[hof]: http://en.wikipedia.org/wiki/Higher-order_function

That may seem simple but has tremendous implications. We are going to explore some of them here.

**Clarification note**: I'm going to use examples written in Ruby. They are meant to be applicable to any programming language that can use functions as arguments and return values. Also, I'm not going to follow some Ruby conventions in favor of having the most homogeneous and explicit examples possible.

Starting point
--------------

```ruby
minimum_age_to_drive = 17
people = [
  {:name => "John", :age => 17},
  {:name => "Jane", :age => 15},
]

for person in people
  puts person[:name] if person[:age] >= minimum_age_to_drive
end
```

This `for` loop, although small and harmless looking, contains highly entangled code. It is full of concreteness. Nothing in it can be reused. Users are going to be forced to duplicate the loop and most of the operations if they want to do something similar.

For example: let's suppose in some parts we want to print the names of the people who can drive. And, in other parts we want to save them in a file. We want to reuse everything but the process of doing something with the names of each eligible driver. What do we do?

```ruby
eligible_driver_names = []
for person in people
  eligible_driver_names << person[:name] if person[:age] >= minimum_age_to_drive
end

#in some part of the code:
for name in eligible_driver_names
  $stdout.puts name
end

#in other part of the code:
for name in eligible_driver_names
  file.puts name
end
```

We have here structural duplication. We are repeating the same loop structure each time we do something with each element of a collection. The only difference between the last two loops is the stream where we `puts` the `name`. We could change those two `for` loops that iterate through the `eligible_driver_names` into something more generic by doing this:

```ruby
puts_eligible_driver_names = lambda{|stream|  
  for name in eligible_driver_names
    stream.puts name
  end
}

#in some part of the code:
puts_eligible_driver_names.call $stdout

#in other part of the code:
puts_eligible_driver_names.call file
```

But that will solve a part of the problem and only for this concrete scenario. We don't want to be tied to `eligible_driver_names` so we can reuse it in more scenarios. We could generalise it a bit more by passing it as a parameter like this:

```ruby
puts_each = lambda{|collection, stream|  
  for element in collection
    stream.puts element
  end
}

#in some part of the code:
puts_each.call eligible_driver_names, $stdout

#in other part of the code:
puts_each.call eligible_driver_names, file
```

That code now is more generic but it is completely tied to the `puts` method on the `stream` object. We want a more generic solution, one that we can reuse in a wider context. We need to be able to do anything with each element of the collection, not only `puts` on a `stream`.

Receive functions as arguments
------------------------------

Because we want to swap the code that we execute on **each** element of a collection, we can send that code in a function as an argument.

```ruby
each = lambda{|collection, side_effect|
  for element in collection
    side_effect.call element
  end
}

#in some part of the code:
each.call eligible_driver_names, lambda{|name| $stdout.puts name}

#in other part of the code:
each.call eligible_driver_names, lambda{|name| file.puts name}
```

*(Again: If you know Ruby, you may be confused that I am defining the `each` function that uses the `for` loop which relies on the method `each` in the collection object. Please keep in mind that these examples are not meant to be Ruby specific. Think about that `for` loop as the lower-level one: `for (i=0; i<collection.size; i+=1) collection[i]`)*

Now, this is `each` function is very generic. It can be reused in a very wide context.

We can do the same with the remaining piece of code from the beginning:

```ruby
eligible_driver_names = []
for person in people
  eligible_driver_names << person[:name] if person[:age] >= minimum_age_to_drive
end
```

To be able to do that we need to separate the parts involved in [each responsibility][srp]. Let's generalise the one that **selects** the drivers first:

[srp]: http://en.wikipedia.org/wiki/Single_responsibility_principle

```ruby
drivers = []
for person in people
  drivers << person if person[:age] >= minimum_age_to_drive
end
```

```ruby
select = lambda{|collection, predicate|
  selected = []
  for element in collection
    selected << element if predicate.call element
  end
  return selected
}

drivers = select.call people, lambda{|person| person[:age] >= minimum_age_to_drive}
```

And now the one that **maps** each driver to his/her name:

```ruby
eligible_driver_names = []
for person in drivers
  eligible_driver_names << person[:name]
end
```

```ruby
map = lambda{|collection, transformation|
  mapped = []
  for element in collection
    mapped << transformation.call(element)
  end
  return mapped
}

eligible_driver_names = map.call drivers, lambda{|person| person[:name]}
```

By hiding the `for` loop inside those functions, we have created an [abstraction][abstraction]. We have separated *what to do* from *how to do it*. Now the clients can depend only on *what to do*. That enables the possibility to change *how to do it* without affecting them.

[abstraction]: http://en.wikipedia.org/wiki/Abstraction_(computer_science)

For example: In the `each` function *what we do* is: *do a `side_effect` with each element in the collection*. *How we do it* is: *iterating sequentially with the `for` loop*.

We could change the implementation of `each` for one that process the elements in parallel to take advantage of multi-core or several clusters. We could also change the implementation of `map` for one that does the transformation only if the result is going to be used ([lazy evaluation][lazy]).

[lazy]: http://en.wikipedia.org/wiki/Lazy_evaluation

Return functions
----------------

Let's imagine we want to `select` the `people` in different ways. Not only by the requirement on age to drive. Let's suppose we also want to `select` the `people` who are called "Jane".

```ruby
drivers = select.call people, lambda{|person| person[:age] >= minimum_age_to_drive}
janes = select.call people, lambda{|person| person[:name] == "Jane"}
```

We are passing the `people` to each one of those calls. If we were only selecting `people`, that may seem redundant. We could define a function that only selects on the `people`.

```ruby
select_people = lambda{|predicate|
  return select.call people, predicate
}

drivers = select_people.call lambda{|person| person[:age] >= minimum_age_to_drive}
janes = select_people.call lambda{|person| person[:name] == "Jane"}
```

But that only works if `people` is defined before `select_people`. Also, it is only reusable in this context because it is tied directly to `people`. We don't want to be obligated to define the collection before. And we would like to do that with any collection.

```ruby
select_collection = lambda{|collection|
  return lambda{|predicate|
    return select.call collection, predicate
  }
}
select_people = select_collection.call people

drivers = select_people.call lambda{|person| person[:age] >= minimum_age_to_drive}
janes = select_people.call lambda{|person| person[:name] == "Jane"}
```

The result of the function `select_collection` is another function with one parameter less. That is called [partial application][partial]. Which means that we are predefining arguments.

[partial]: http://en.wikipedia.org/wiki/Partial_application

Object orientation
------------------

At this point you may be thinking: All of that is *functional programming*. It is not *object oriented*. They are two different paradigms. We have been nesting function calls. We have been sending the collection around and transforming it. We haven't defined any class!

In object oriented you have the data and the behaviour that uses that data together in a class. The functions `each`, `select`, `map` operate over a collection. So that behaviour could be in the collection class.

```ruby
class Collection
  def initialize(collection)
    @collection = collection
  end

  def each(side_effect)
    for element in @collection
      side_effect.call element
    end
    return self
  end

  def select(predicate)
    selected = []
    for element in @collection
      selected << element if predicate.call element
    end
    return Collection.new(selected)
  end

  def map(transformation)
    mapped = []
    for element in @collection
      mapped << transformation.call(element)
    end
    return Collection.new(mapped)
  end
end

eligible_driver_names = Collection.new(people)
  .select(lambda{|person| person[:age] >= minimum_age_to_drive})
  .map(lambda{|person| person[:name]})

#in some part of the code:
eligible_driver_names
  .each(lambda{|name| $stdout.puts name})

#in other part of the code:
eligible_driver_names
  .each(lambda{|name| file.puts name})
```

This is an emulation of how some object oriented languages implement them. In Ruby: [`select`][select], [`map`][map], [`each`][each].

[select]: http://ruby-doc.org/core-2.2.1/Enumerable.html#method-i-select
[map]: http://ruby-doc.org/core-2.2.1/Enumerable.html#method-i-map
[each]: http://ruby-doc.org/core-2.2.0/Array.html#method-i-each

Let's look at the `initialize` method. It is the class constructor. It is a function that returns an instance of that class with the constructor argument `collection` predefined. It is doing [partial application][partial]. It enables the users of the methods `each`, `select` and `map` to use them without having to send the `collection` argument.

Once you have an instance, you can call to any of its methods. In a way, you can think of an instance as a container of functions that operate with the arguments that have been passed to the constructor and any extra arguments passed to the function itself.

Given that idea we can express what we know as object oriented only with [higher-order functions][hof]:

```ruby
collection_initialize = lambda{|collection|
  _self = {
    :each => lambda{|side_effect|
      for element in collection
        side_effect.call element
      end
      return _self
    },

    :select => lambda{|predicate|
      selected = []
      for element in collection
        selected << element if predicate.call element
      end
      return collection_initialize(selected)
    },

    :map => lambda{|transformation|
      mapped = []
      for element in collection
        mapped << transformation.call(element)
      end
      return collection_initialize(mapped)
    },
  }
  return _self
}

eligible_driver_names = collection_initialize.call(people)[:select]
  .call(lambda{|person| person[:age] >= minimum_age_to_drive})[:map]
  .call(lambda{|person| person[:name]})

#in some part of the code:
eligible_driver_names[:each].call(lambda{|name| $stdout.puts name})

#in other part of the code:
eligible_driver_names[:each].call(lambda{|name| file.puts name})
```

Is this really that different from the `Collection` class?

A small peek into monads
------------------------

Notice how at the end of `select` and `map` methods, we are returning the results wrapped in another instance of `Collection`. And at the end of `each` we are returning `self` (because we haven't modified the collection). We need to do that if we want to keep using this behaviour with the results. If we don't do that, we can only call once to those methods, we cannot [chain calls][chaining] like this:

[chaining]: http://en.wikipedia.org/wiki/Method_chaining

```ruby
Collection.new(people)
  .select(lambda{|person| person[:age] >= minimum_age_to_drive})
  .map(lambda{|person| person[:name]})
  .each(lambda{|name| $stdout.puts name})
  .each(lambda{|name| file.puts name})
```

We transformed the syntax of nesting function calls into [method chaining][chaining]. We combined multiple functions into a single statement. And you may have not realised yet, but `Collection` is a [monad][monad]. Don't get intimidated by that name. That's another fancy name like *[higher-order functions][hof]* or *[partial application][partial]*. We might explore a lot deeper into that concept in future blog posts.

[monad]: http://en.wikipedia.org/wiki/Monad_(functional_programming)
