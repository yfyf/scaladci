# Scala DCI

The [Data Context Interaction (DCI)](http://en.wikipedia.org/wiki/Data,_context_and_interaction) 
paradigm by Trygve Reenskaug and James Coplien embodies true object-orientation where
runtime Interactions between a network of objects in a particular Context 
is understood _and_ coded as first class citizens. ScalaDCI tries to get as close as Scala allows
us to realize the goals of DCI by using macro source code transformation, also known as "injectionless DCI".

DCI is a new paradigm that implies many layers of new thinking that are fundamentally different from
how mainstream "object-oriented" coding is perceived Today. It's not just a "technique" or a "pattern". It
is also about how we think about the domain and what the system is versus what the system does. And DCI
offers new ways to be more explicit about the does-side in ways that enable us to reason about
what the code will do at runtime. Something that the Gang of Four admitted most "object-oriented" systems
don't offer with their deeply nested deferred hierarchies of classes organized in endless patterns that
tell almost nothing about the path _objects_ instantiated from those classes will take at runtime. 
That unfortunately makes Todays programs almost impossible to reason about and therefore completely reliant on 
extensive test suites that desperately try to catch runtime surprises - surprises that DCI tries to avoid 
from the outset by taking a fundamentally more reasonable approach.

At the moment extensive research and development is taken on by James Coplien to develop a truly 
object-oriented research language called "trygve", named after the inventor of DCI, Trygve Reenskaug. 
See more on the 
[official DCI site](http://www.fulloo.info) and on the
[google list](https://groups.google.com/forum/#!forum/object-composition).


To see how ScalaDCI works, let's take a very simple example of a Data class Account with some basic methods:
```scala
case class Account(name: String, var balance: Int) {
  def increaseBalance(amount: Int) { balance += amount }
  def decreaseBalance(amount: Int) { balance -= amount }
}
```
This is a primitive data class only knowing about its own data and how to manipulate that. 
The concept of a transfer between two accounts we can leave outside its responsibilities and instead 
delegate to a "Context" - the MoneyTransfer Context class. In this way we can keep the Account class 
very slim and avoid that it gradually takes on more and more responsibilities for each use case 
it participates in.

Our Mental Model of a money transfer could be "Withdraw amount from a source account and deposit the 
amount in a destination account". Interacting concepts like our "source"
and "destination" accounts we call "Roles" in DCI. And we can define how they can interact in our
Context to accomplish a money transfer:
```Scala
@context
class MoneyTransfer(source: Account, destination: Account, amount: Int) {

  source.withdraw // Interactions start...

  role source {
    def withdraw() {
      source.decreaseBalance(amount)
      destination.deposit
    }
  }

  role destination {
    def deposit() {
      destination.increaseBalance(amount)
    }
  }
}
```

We want our source code to map as closely to our mental model as possible so that we can confidently and easily
overview and reason about _how the objects will interact at runtime_! We want to expect no surprises at runtime. 
With DCI we have all runtime interactions right there! No need to look through endless convoluted abstractions, 
tiers, polymorphism etc to answer the reasonable question _where is it actually happening, goddammit?!_

At compile time, our @context macro annotation transforms the abstract syntax tree (AST) of our code to enable our
_runtime data objects_ to "have" those extra Role Methods. Well, I'm half lying to you; the 
objects won't "get new methods". Instead we call Role-name prefixed Role methods that are
lifted into Context scope which accomplishes what we intended in our source code. Our code gets transformed as 
though we had written this:

```Scala
class MoneyTransfer(source: Account, destination: Account, amount: Int) {
  
  source_withdraw()
  
  private def source_withdraw() {
    source.decreaseBalance(amount) // Calling Data instance method
    destination_deposit()          // Calling Role method
  }
  
  private def destination_deposit() {
    destination.increaseBalance(amount)
  }
}
```

## Role methods that "shadow" instance methods
In ScalaDCI, role methods will always take precedence over instance methods of the object that will play the role.

Generally we want to avoid defining a role method with a name that clashes with - or "shadows" - an instance method since this could cause confusion about the runtime behavior of our program. "Which method is called on the role-playing object?".

Therefore ScalaDCI tries to determine the instance type to see if a role method shadows one of its  methods, and if so throw a compile time error.

But at runtime we might not be able to determine the instance type and thereby what instance method it has. In that case ScalaDCI falls back on always calling the role method. In this case no error is thrown.


## `self` reference to a Role Player
As an alternative to using the Role name to reference a Role Player we can also use `self`:
```Scala
  role source {
    def withdraw {
      self.decreaseBalance(amount)  
      destination.deposit
    }
  }

  role destination {
    def deposit {
      self.increaseBalance(amount)
    }
  }
```
or `this`:
```Scala
  role source {
    def withdraw {
      this.decreaseBalance(amount)
      destination.deposit
    }
  }

  role destination {
    def deposit {
      this.increaseBalance(amount)
    }
  }
```


## Multiple roles
We can "assign" or "bind" a domain object to several Roles in our Context by simply making
more variables with role names pointing to that object:
```Scala
@context
class MyContext(someRole: MyData) {
  val otherRole = someRole
  val localRole = new DummyData()
  
  someRole.foo() // prints "Hello world"
  
  role someRole {
    def foo() {
      someRole.doMyDataStuff()
      otherRole.bar()
    }
  }
  
  role otherRole {
    def bar() {
      localRole.say("Hello")
    }
  }
  
  role localRole {
    def say(s: String) {
      println(s + " world")
    }
  }
}
```
As you see in line 3, otherRole is simply a reference pointing to the MyData instance (named someRole).

Inside each role definition we can still use `self`/`this`.

We can add as many references/role definitions as we want. This is a way to 
allow different Roles of a Use Case each to have their own meaningful namespace for defining their 
role-specific behavior / role methods.

## How does it work?
In order to have an intuitive syntax like

```scala
role roleName {
  // role methods...
}
```

for defining a Role and its Role methods we need to make a Scala
contruct that is valid before our macro annotation can start transforming our code:

```scala
object role extends Dynamic {
  def applyDynamic(obj: Any)(roleBody: => Unit) = roleBody
}
```

Since the `role` object extends the `Dynamic` marker trait and we have defined an
 `applyDynamic` method, we can invoke methods with arbitrary method names on the
 `role` object. When the compiler find that we are trying to call a method on
  `role` that we haven't defined (it doesn't type check), it will rewrite our code 
  so that it calls `applyDynamic`:
 
```scala
role.foo(args) ~~> role.applyDynamic("foo")(args)
role.bar(args) ~~> role.applyDynamic("bar")(args)
```

For the purpose of DCI we can presume to call a method on `role` that "happens"
to have a Role name:

```scala
role.source(args)      ~~> role.applyDynamic("source")(args)
role.destination(args) ~~> role.applyDynamic("destination")(args)
```

Scala allow us to replace the `.` with a space and the parentheses with curly
braces:

```scala
role source {args}      ~~> role.applyDynamic("source")(args)
role destination {args} ~~> role.applyDynamic("destination")(args)
```

You see where we're getting at. Now, the `args` signature in our
`applyDynamic` method has a "by-name" parameter type of `=> Unit` that
 allow us to define a block of code that returns nothing:
 
```scala
role source {
  doThis
  doThat
}      
~~> role.applyDynamic("source")(doThis; doThat) // pseudo code
```

The observant reader will note that "source" given the Dynamic invocation
capability is merely a "free text" name that has no connection to the object 
that we have called "source":

```scala
val source = new Account(...) // `source` is an object identifier
role source {...}             // "source" is a method name
```

In order to enforce that the method name "source" points to the object `source`
our `@context` macro annotation checks that the method name has a corresponding 
identifier in the scope of the annotated Context. If it doesn't it won't compile 
and the programmer will be noticed of available identifier names (one could have 
misspelled the Role name for instance).


## Scala DCI demo application

In the [Scala DCI Demo App](https://github.com/DCI/scaladci-demo) you can see an example of
how to create a DCI project.


## Using Scala DCI in your project

ScalaDCI is available for Scala 2.11.6 at [Sonatype](https://oss.sonatype.org/content/repositories/releases/org/scaladci/scaladci_2.11/0.5.5/).
To start coding with DCI in Scala add the following to your SBT build file:

    libraryDependencies ++= Seq(
      "org.scaladci" %% "scaladci" % "0.5.5"
    ),
    addCompilerPlugin("org.scalamacros" % "paradise" % "2.1.0-M5" cross CrossVersion.full)


## Building Scala DCI
```
git clone https://github.com/DCI/scaladci.git
<open in your IDE>
```


Have fun!

Marc Grue<br>
April 2015

### DCI resources
Discussions - [Object-composition](https://groups.google.com/forum/?fromgroups#!forum/object-composition)<br>
Website - [Full-OO](http://fulloo.info)<br>
Wiki - [DCI wiki](http://en.wikipedia.org/wiki/Data,_Context,_and_Interaction)

### Credits
Trygve Renskaug and James O. Coplien for inventing and developing DCI.

Scala DCI solution inspired by<br>
- Risto Välimäki's [post](https://groups.google.com/d/msg/object-composition/ulYGsCaJ0Mg/rF9wt1TV_MIJ) and<br>
- Rune Funch's [Marvin](http://fulloo.info/Examples/Marvin/Introduction/) DCI language 
and [Maroon](http://runefs.com/2013/02/14/using-moby-to-do-injectionless-dci-in-ruby/) for Ruby.
