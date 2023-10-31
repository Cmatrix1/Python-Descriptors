
![safjsd](https://github.com/Cmatrix1/Python-Descriptors/assets/74909796/de5bce1d-802e-4942-98bf-faf121896c4d)

# Python Descriptors
Complete Tuturial of Python Descriptors

## Introduction

Welcome to the intriguing world of Python descriptors! This article may seem a bit complex, but trust me, understanding descriptors is absolutely worth it.

They unlock the inner workings of properties, methods, slots, and even functions, providing a solid foundation for Python mastery. 

In this section, we'll primarily focus on two types of descriptors: non-data descriptors and data descriptors. These two types behave slightly differently, and we'll delve into their distinctions. 

As we progress, we'll learn how to craft and utilize custom data descriptors. You'll discover the unique advantages and practical applications of writing your own data descriptors. 

Additionally, we'll navigate through some common pitfalls related to storing data associated with data descriptors. By learning how to avoid these pitfalls, you'll enhance your descriptor implementation skills. 

As we explore deeper into this section, we'll tackle advanced topics such as weak references and their relationship to weak dictionaries. These concepts will broaden your understanding and empower you to create more robust and efficient code. 😄 There's no denying that this article is packed with captivating content. So, without further ado, let's dive in and unravel the mysteries of Python descriptors together! Thank you for joining me.


## Problem We Want to Solve
Suppose we want a User class whose age must always be integer and name must always be string

Of course we can use a property with getter and setter methods

Let's implement it with propertys first:
```python
class User:
    @property
    def name(self):
        return self._name
    
    @name.setter
    def name(self, value):
        self._name = str(value)

    @property
    def age(self):
        return self._age
    
    @age.setter
    def age(self, value):
        self._age = int(value)
```
this is tedious, repetitive boiler plate code! **better way needed!**

# Descriptors

And this is where the descriptor protocol comes in !

Python descriptors are simply objects that implement the descriptor protocol.

The protocol is comprised of the following special methods - not all are required.

- `__get__`: retrieve the property value
- `__set__`: store the property value
- `__del__`: delete a property from the instance
- `__set_name__`:  capture the property name as it is being defined.

There are two types of Descriptors:

1. Non-data descriptors: these are descriptors that only implement `__get__` (and optionally `__set_name__`)
2. Data descriptors: these implement the `__set__` method, and normally, also the `__get__` method.

#### Let's create a simple non-data descriptor:
```python
from datetime import datetime

class TimeUTC:
    def __get__(self, instance, owner_class):
        return datetime.utcnow().isoformat()
```

So `TimeUTC` is a class that implements the `__get__` method only, and is therefore considered a non-data descriptor.

We can now use it to create properties in other classes:
```python
class Logger:
    current_time = TimeUTC()
```
---
**NOTE** :
That `current_time` is a class attribute:
```python
Logger.__dict__
```
```bash
mappingproxy({'__module__': '__main__',
              'current_time': <__main__.TimeUTC at 0x7fdcd84bbd68>,
              '__dict__': <attribute '__dict__' of 'Logger' objects>,
              '__weakref__': <attribute '__weakref__' of 'Logger' objects>,
              '__doc__': None})
```
---


We can access that attribute from an instance of the `Logger` class:
```python
l = Logger()
l.current_time
'2023-10-27T08:11:58.319429'
```

We can also access it from the class itself, and for now it behaves the same (we'll come back to that later):
```python
Logger.current_time
'2023-10-27T08:11:58.327000'
```


#### Let's consider another example.
Suppose we want to create class that allows us to select a random status and random favorite color for the user instance.
We could approach it this way:
```python
from random import choice, seed

class Person:
    @property
    def status(self):
        return choice(('😃', '😄', '😊', '😉', '😍', '🤩', '😎', '🥳', '😇', '🙌'))
        
    @property
    def favorite_color(self):
        colors = ('🔴', '🟠', '🟡', '🟢', '🔵', '🟣', '🟤', '⚫', '⚪', '🌈')
        return choice(colors)

```

This was pretty easy, but as you can see both properties essentially did the same thing - they picked a random choice from some iterable.
Let's rewrite this using a custom descriptor:

```python
class Choice:
    def __init__(self, *choices):
        self.choices = choices
        
    def __get__(self, instance, owner_class):
        return choice(self.choices)

```
And now we can rewrite our `Person` class this way:

```python
class Person:
    status = Choice('😃', '😄', '😊', '😉', '😍', '🤩', '😎', '🥳', '😇', '🙌')
    favorite_color = Choice('🔴', '🟠', '🟡', '🟢', '🔵', '🟣', '🟤', '⚫', '⚪', '🌈')
```

Of course we are not limited to just cards, we could use it in other classes too:
```python
class Dice:
    die_1 = Choice(1,2,3,4,5,6)
    die_2 = Choice(1,2,3,4,5,6)
    die_3 = Choice(1,2,3,4,5,6)
```

## Getters and Setters

### Getters
So far we have seen how the `__get__` method is called when we assign an instance of a descriptors to a class attribute.
But we can access that attribute either from the class itself, or the instance - as we saw in the last lecture, both accesses end up calling the `__get__` method.
So, when __get__ is called, we may want to know:
-  which instance was used (if any) ->  None if called from class
-  what class owns the TimeUTC (descriptor) instance -> Logger in our case

this is why we have the signature: `__get__(self, instance, owner_class)`
So we can return different values from __get__ depending on:
- called from class
- called from instance

```python
from datetime import datetime

class TimeUTC:
    def __get__(self, instance, owner_class):
        print(f'__get__ called, self={self}, instance={instance}, owner_class={owner_class}')
        return datetime.utcnow().isoformat()

class Logger1:
    current_time = TimeUTC()
```
Now let's access `current_time` from the class itself:
```python
Logger1.current_time
'__get__ called, self=<__main__.TimeUTC object at 0x0000026BF3C87210>, instance=None, owner_class=<class '__main__.Logger1'>'
'2023-10-27T09:58:00.586792'
```
As you can see, the `instance` was `None` - this was because we called the descriptor from the `Logger1` class, not an instance of it. The `owner_class` tells us this descriptor instance is defined in the `Logger1` class.

But if we call the descriptor via an instance instead:
```python
l1 = Logger1()
l1.current_time
'__get__ called, self=<__main__.TimeUTC object at 0x0000026BF3C87210>, instance=<__main__.Logger1 object at 0x0000026BF41DEC50>, owner_class=<class '__main__.Logger1'>'
```


### very often, we choose to:
- return the descriptor (TimeUTC) instance when called from class itself (Logger class) gives us an easy handle to the descriptor instance
- return the attribute value when called from an instance of the class

```python
from datetime import datetime

class TimeUTC:
    def __get__(self, instance, owner_class):
        if instance is None:
            # called from class
            return self
        else:
            # called from instance
            return datetime.utcnow().isoformat()

class Logger:
    current_time = TimeUTC()

Logger.current_time
'<__main__.TimeUTC at 0x26bf41df350>'

l = Logger()
l.current_time
'2023-10-27T09:58:00.644233'
```

This is consistent with the way properties work:
```python
class Logger:
    @property
    def current_time(self):
        return datetime.utcnow().isoformat()

Logger.current_time
'<property at 0x26bf420c950>'
```

This returned the property instance, whereas calling it from an instance:
```python
l = Logger()
l.current_time
'2023-10-27T09:58:00.661615'
```

#### Now, there is one subtle point we have to understand when we create multiple instances of a class that uses a descriptor as a class attribute.
Since the descriptor is assigned to an **class attribute**, all instances of the class will **share** the same descriptor instance!

```python
class TimeUTC:
    def __get__(self, instance, owner_class):
        if instance is None:
            # called from class
            return self
        else:
            # called from instance
            print(f'__get__ called in {self}')
            return datetime.utcnow().isoformat()
        
class Logger:
    current_time = TimeUTC()

l1 = Logger()
l2 = Logger()
```
But look at the `current_time` for each of those instances

```python
l1.current_time is l2.current_time
> True
```

As you can see the **same** instance of `TimeUTC` was used.
This does not matter in this particular example, since we just return the current time, but watch what happens if our property relies on some kind of state in the descriptor:

```python
class Countdown:
    def __init__(self, start):
        self.start = start + 1
        
    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            self.start -= 1
            return self.start

class Rocket:
    countdown = Countdown(10)
```
Now let's say we want to launch two rockets:

```
rocket1 = Rocket()
rocket2 = Rocket()
```
And let's start the countdown for each one:
```python
rocket1.countdown
> 10
rocket2.countdown
> 9
rocket1.countdown
> 8
```

As you can see, the current countdown value is shared by both `rocket1` and `rocket2` instances of `Rocket` - this is because the `Countdown` instance is a class attribute of `Rocket`. So we have to be careful how we deal with instance level state.


### Setters
The `__set__` method works in a similar way to `__get__` but it is used when we assign a value to the class attribute.
`__set__` signature is as follows: self, instance, value
- instance: the instance the __set__ method was called from
- value: the value we want to assign to the attribute

You'll notice there is no owner_class like we have in the `__get__` method
setters (and deleters) are always called from instances

```python
class IntegerValue:
    def __set__(self, instance, value):
        print(f'__set__ called, instance={instance}, value={value}')
        
    def __get__(self, instance, owner_class):
        if instance is None:
            print('__get__ called from class')
        else:
            print(f'__get__ called, instance={instance}, owner_class={owner_class}')

class Point2D:
    x = IntegerValue()
    y = IntegerValue()

p = Point2D()
p.x = 100
> '__set__ called, instance=<__main__.Point2D object at 0x000001E7564084D0>, value=100'
```

## Caveat with Set and Delete (and Get)
Notice that we have only created a single instance of the `TimeUTC` descriptor:
```python
class Logger:
current_time = TimeUTC()
```
So what happens when we do this?
```python
l1 = Logger()
l2 = Logger()
```
Any instance of `Logger` will be referencing the same instance of `TimeUTC` in this case it does not matter because `__get__` just returns the current UTC time
But what happens when we have to "store" and "retrieve" data from the instances?
Suppose `IntegerValue` is a data descriptor ->  implements `__get__` and `__set__` methods
```python
class Point2D:
    x = IntegerValue()
    y = IntegerValue()
# two separate instances of IntegerValue assigned to the class attributes x and y

p1 = Point2D()
p2 = Point2D()
# two separate instances of Point2D
```
But what object does p1.x reference? -> the class attribute x
what about p2.x? -> the same class attribute x (the same instance of IntegerValue)

 we have to be mindful of which instance we are "storing" the data for
this is one of the reasons both __get__ and __set__ need to know the **instance**

## Storing
So, where should we store the values `x` and `y` in the previous example? 

### Approach 1
Many "tutorials" I see on the web naively store the value in the descriptor itself:
```python
class IntegerValue:
    def __set__(self, instance, value):
        self._value = int(value)
        
    def __get__(self, instance, owner_class):
        if instance is None:
            return self
        else:
            return self._value

class Point2D:
    x = IntegerValue()
    y = IntegerValue()
```
At first blush, this seems to work just fine:
```python
p1 = Point2D()
p2 = Point2D()

p1.x = 1.1
p1.y = 2.2

print(p1.x, p1.y)
> (1, 2)
```
But, remember the point I was making about the instance of the descriptor (`IntegeraValue` in this case) being shared by all instances of the class (`Point2D` in this case)?
```python
p2 = Point2D()
print(p2.x, p2.y)
> (1, 2)
```
And of course if we set the value:
```python
p2.x = 100.9
print(p2.x, p1.x)

> (100, 100)
```
So, obviously using the descriptor instance dictionary for storage at the instance level is probably not going to work in most cases!
And this is the reason both the `__get__` and `__set__` methods need to know which instance we are dealing with.

### Approach 2
We are going to **Assuming** that the `instance` is a hashable object, and use a dictionary in the descriptor to store instance specific values.
```python
class IntegerValue:
    def __init__(self):
        self.values = {}
        
    def __set__(self, instance, value):
        self.values[instance] = int(value)
        
    def __get__(self, instance, owner_class):
        if instance is None:
            return self
        else:
            return self.values.get(instance)
```

```python
class Point2D:
    x = IntegerValue()
    y = IntegerValue()

p1 = Point2D()

p1.x = 10.1
p1.y = 20.2

p1.x, p1.y
> (10, 20)
```
In fact, we can see the dictionary in the descriptor instances:
```python
Point2D.x.values
> {<__main__.Point2D at 0x212d93d9710>: 10}
Point2D.y.values
> {<__main__.Point2D at 0x212d93c2110>: 20}

```
where the key in both of these is our `p1` object
We can now create a second point, and go through the same steps:
```python
p2 = Point2D()
p2.x = 100.1
p2.y = 200.2

Point2D.x.values
> {<__main__.Point2D at 0x212d93b2290>: 10,
> <__main__.Point2D at 0x212d931c8d0>: 100}

Point2D.y.values
> {<__main__.Point2D at 0x212d93b2290>: 20,
> <__main__.Point2D at 0x212d931c8d0>: 200}
```
And everything works just fine ( Or does it?? ):
```bash
p1.x, p1.y, p2.x, p2.y
> (10, 20, 100, 200)
```
We actually have a potential memory leak - notice how the dictionary in the desccriptor instance is **also** storing a reference to the point object - as a **key** in the dictionary.

---
Let's write a simple utility function that allows us to get the reference count for an object given it's id (and it only makes sense if the id we use still has a valid non-destroyed object):
```python
import ctypes

def ref_count(address):
    return ctypes.c_long.from_address(address).value

p1 = Point2D()
id_p1 = id(p1)

ref_count(id_p1)
> 1
```
Now let's set the `x` property of `p1`:
```python
p1.x = 100.1
```
And let's check the ref count again:
```python
ref_count(id_p1)
> 2
```
As you can see it's now `2`. if we delete our main reference to `p1` that is in our global namespace:
```python
'p1' in globals()
> True
del p1
'p1' in globals()
> False
ref_count(id_p1)
> 1
```
And our reference count is still `1`, which means the object itself has not been destroyed!
In fact, we can see that object referenced in our data descriptor dictionary:
```python
Point2D.x.values.items()
> dict_items([(<__main__.Point2D object at 0x00000212D93B2290>, 10), (<__main__.Point2D object at 0x00000212D931C8D0>, 100), (<__main__.Point2D object at 0x00000212D9390150>, 100)])
```
As you can see, the last element's key is the same id as what `p1` was referencing.
So, although we deleted `p1`, the object was not destroyed - this can result in a memory leak.
There are a few ways we can handle this issue. The first one we are going to look at is something called **weak references**. So let's segway into that next.



