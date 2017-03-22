---
layout: post
title:  "C# Implicit Typing"
date:   2017-03-21 07:30:00 -0500
categories: programming c#
---

### What is it?

Implicit typing is a language feature of C# which allows you to declare variables without
explicitly stating the type. Here is an example of an explicit type declaration followed by
an implicit type declaration:

```csharp
string firstName = "Jim";
```

```csharp
var firstName = "Jim";
```

The var keyword allows for a cleaner line of code without losing or obscuring the intent. 
We still know we are creating the string "Jim" and that we can access it using "firstName".

The benefits of this are evident if you consider a typical block of C# code, where we are
newing up a bunch of objects. It typically has a kludgy cluttered look to it:

```csharp
BusinessObject businessObject = new BusinessObject();
OtherBusinessObject otherBusinessObject = new OtherBusinessObject();
BusinessObjectProvider businessObjectProvider = new BusinessObjectProvider();
```

The explicit type declaration here is un-necessary and harder to read. We can use var again
to clean things up:

```csharp
var businessObject = new BusinessObject();
var otherBusinessObject = new OtherBusinessObject();
var businessObjectProvider = new BusinessObjectProvider();
```

Just like the previous example, the code is cleaner and we have not lost any information.

### Can I use it everywhere?

You can only use it to declare a variable if you are initializing the variable in the same
line. That way the compiler can infer the type and convert it behind the scenes to an explicit
declaration. The following lines will raise compiler errors because the compiler can't possibly 
know what type "someObject" is:

```csharp
var someObject;
var someObject = null;
```

It's important to note that C# is still a strongly typed language, and by use of the var
keyword we are not declaring dynamically typed objects.

### When should I use it or not use it?

So far this feature sounds like a dream. Reduced code verbosity while maintaining the usual
compile time type checks, what could possibly go wrong?

Well, there are ways in which this feature can be misused or overused.

You should use it when the assignment (things to the right of the equals sign) makes it obvious
what the type of the variable is. That way you gain all the benefits of cleaner, less cluttered
code without losing any information.

```csharp
var someObject = someObjectFactory.Create();
```

This is an example of implicit typing being misused. We don't know what type 
"someObject" is by just reading this line of code. The gain in readability is offset by the loss 
of information. In this case we should explicitly declare the variable so it is clear we are 
declaring an object of type "SomeObject".

```csharp
SomeObject someObject = someObjectFactory.Create();
```

### Thoughts

The Java language designers considered the possibility of the above case so unacceptable that
they have avoided implementing implicit typing at all. I think that's a little too conservative
and Java desperately needs features like this to reduce it's verbosity.

As small and inconsequential this feature is, it's one of many reasons why C# is much more 
pleasant to work with than Java.



