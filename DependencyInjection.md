# Dependency Injection and Functional Programming in Java and a glimpse of Kotlin

I've spent most of my career doing developments in an imperative style (and if we go further back in time, even in "procedural" style). For the last years, the _functional Programming_ paradigm is becoming more and more popular for it's many benefits:

 * It allows us to define _what_ we want in a more concise way, instead of _how_ we want it.
 * We can define _pure functions_, which take some parameters as an input and *always* return the same output for an input given. No Exception handling and no side effects, which is great for Multi-Thread, among other things.
 * The use of "pure funtions" allows us to deine our business logic as a series of functions which can be applied in sequence, to transform the data, in a similar fashion as Streams of Pipes.


Another technique that we are all very familiar with and it's become a standard is _Dependency Injection_: In a few words, it's the possibility to define the behaviour of our system in an external way: we just define the contract/signature of our modules, and the concrete implementation is "injected" at the beginning of the lifecycle. If you are using _Spring_, you are using dependency injection.

In Java, you can use Spring by just adding a few annotations. Here we have 2 versions of the same Class ("Example"), which has a dependency to a Repository:

```
// First version: Setter injection
@Component
class Example {

@Autowired
Repository repo;

public Example() {}
}

...

// Second version: Constructor injection
@Component
class Example {

Repository repo;

@Autowire
public Example(Repository repo) {
this.repo = repo;
}

```

with _Spring_, dependencies can be injected either through the Constructor (when the instance is created) or through _setters_ (at a later point in time, but before the instance is used)

_Spring_ does this "injection magic" for us, using a technique called "Assembly Injection", which consists of parsing the classes in our Project and injecting the dependencies where they are found.

# But is Java a good language for Functional Programming?


_Java_ is an _Object Oriented_ language, that means that everything must be implemented in a Class. 

These classes can implement some functions (_methods_), and also store information in fields (which we call the _state_ of that object). 

So here is where the problem is: If those methods have side effects (which they have many times), then the state of the object might change between different calls to the methods, so we can not be sure anymore that the output is gonna be the same. 

An additional problem comes when you apply multi-threading to the picture: different threads callingthose methods at the same time, and those methods in turn modifying the fields (State) of the object.. that leads to _race conditions_ which forces us to make use of shynchronization tecniques, etc.


So lets see how we can change slightly our classes and follow some practices in order to make our Java code more functional-style. After that we'll be ready to deal with the "dependency injection" problem.


Here is a Class with one method that uses a Repository to get some Items (from a DB or similar) and makes some changes over them (adding a suffix):

```
class Example {
 Repository repo = ...;
 String suffix = ...
 
 List<String> transformItems() {
 	return repo.loadItems().stream().map (i -> i + suffix)
 	.collect(Collectors.toList());
 }
 
}

```

In this example, the method is using the State of the Object for its purpose (the "repo" instance for loading the items, and the "suffix" field). Even if it's not making any changes to them, it's a potential source of problems. 

## making Java more functional

If we want to deal with the problem highligthed in the previous chapter, 2 options comes to the rescue here:

### Option A: A compromise: we still use "state", but we move as much as we can to the method signatures.

In this approach, we keep our dependencies as part of the State, but we move everything else to the methd signature:

```
class Example {
 Repository repo = ...;
 List<String> transformItems(String suffix) {
 	return repo.loadItems().stream().map (i -> i + suffix)
 	.collect(Collectors.toList());
 }
 
}

```
This is not bad. The _State_ now only contains the dependecy, which can be injected by _Spring_ when creating the instance. For the rest of the class, is only a "container" of Functions.

This the best approach if your are using Java, in my opinion. If you are instead using Kotlin and want to implement a pure Functional programming, please keep reading.

### Option B: Remove the State and move EVERYTHING to the Method

If we remove all the State, we have to put that information somewhere else, and our method is the only place we have: and we only have 2 places where to put it:

 * in the parameter list, as as additional parameter
 * in the Return type.

This option will explained in detail later on, but for now suffice to say that the consequence of removing the state is to make our method signature more complex, since we need to add the dependency there.

## Let's go Pure Functional (Kotlin style).

"Pure Functional" means that we use Functions, not Objects. This is not possible in Java as we know, but it is in Kotlin. We are not diving into Kotlin  here since my experience is limited, but I'll try to address the question of dependecy injection in Kotlin, by trying to simulate a similar scenario in Java.

What scenario can we do in Java that might ressemble a scenario similar to Kotlin using Pure Functions? The answer is when we have to implement our own Dependency Injection (Without using _Spring_)
 

### Pure Dependency Injection (without Spring)

So, in order to put purselves into the Kotlin shoes, we are going to remove _Spring_ from the picture, and we'll figure out ways to solve this dependency injection problem, implementing our own solution for it. This is what is commonly called "Pure Dependency Injection".


So let's take a look at our Class again:

```
class Example {
 Repository repo;
 
 List<String> transformItems(String suffix) {
 	return repo.loadItems().stream().map (i -> i + suffix)
 	.collect(Collectors.toList());
 }
 
}

```

And since we are now going for a more "pure" functional style, we'll remove ALL the State of our Object and move EVERYTHING to our method.

By using the logic and common sense, we only have 2 options here:

 * Inject it as an additional parameter
 * Inject it in the Response


#### Injecting the dependencies in the Parameters list
This is the most easy-to-understand approach, and quite sttraightforward:

```
List<String> transformItems(Repository repo, String suffix) {
 	return repo.loadItems().stream().map (i -> i + suffix)
 	.collect(Collectors.toList());
 }

```
Basically, what we are doing here is to move the responsability of the injection to the "caller". And at the same time, we are making the method signature a bit more complicated, adding more parameters to it.

This is an example of use of this function:

```
Repository repo = new ...
Example object = new Example();
List<String> items = object.transformItems(repo, "processed");
```

#### Injecting the dependencies in the Result Type

This is not as easy to understand. It uses a technique called _deferred execution_ and what it basically does is: Instead of a Function returning a Value, we have a Function returning another Function that returns the value.

Kind of messy.

Leet's see how it works for our function. This is (again) the version using the "parameter" option explained in the previous chapter:

```
List<String> items = object.transformItems(repo, "processed");
```

And this is the way we are going to implement in this new technique:

```
List<String> items = object.transformItems("processed").using(repo);
```

It might look similar to the other one. After all, we are also moving the responsability of the injection to the caller, the difference here is that the dependency is injected in the Result. If you loook at it, the result is not a Value anymore, but a Function with a method called _using_. And that's in that method when we make the injection.

In order to implement this, we need first to define the Function that our method is going to Return:


```
@FunctionalInterface
interface ResultDI<D, R> {
    R using(D dependency);
}
```

This function (_ResultDI_ stands for "Result with Dependency Injection"), takes one parameter (D) and returns a Result (R). the parameter "D" is the dependency we want to inject, and "R" is the final Result.

And now, we modify our "transformItems" method, so it returns this Function instead of a value:

```
public ResultDI<Repository, List<String>> transformItems(String suffix) {
        return repo -> {
            return repo.loadItems().stream().map( i -> i + suffix).collect(Collectors.toList());
        };
    }

```

So now, we managed to remove the injection from the parameter list, and place it in the Return type instead.

## The Flow Chain and where to Implement the Dependency Injection


The previous options for injecting the dependency into a method are different in the place they put it (in the parameter list, or in the Return Type). In both cases, we still have some issues, and they are related to the place _where_ to do the Injection (that is, where we instantiate the dependencies)

In both cases, the injection takes place in the "caller". 
But in most cases we have lots of classes making calls between them, so its not clear where to do it.

Let's imagine the following scenario:

We have 3 very simple classes: Each one of them has a dependency on a Different Repository:

```
Class A {
RepositoryA repoA;
}

Class B {
RepositoryB repoB;
}

Class C {
RepositoryC repoC;
}

```
and imagine that each class has a method "mehodA/B/C" so we have these Flow Chain:

```
 ClassA.methodA() -> ClassB.methodB(...) -> ClassC.methodC(...)
```

So here we have a _ClassA_ that makes a call to a method in _ClassB_ which in turn makes a call to the _ClassC_. 

Here we need to decide where to implement the injection. In other words: Where do we create the instances? We already know that the responsability is in the caller, but here we can see how the scenario gets complicated:

 * When _ClassB_ calls _ClassC_, it's clear that the "caller" here is _ClassB_. So It's up to _ClassB_ to instantiate the _RepositoryC_ that _ClassC_ needs and inject it as either a parameter or in the Return type, as we saw.
 * when  _ClassA_ calls _ClassB_, _ClassA_ is acting as the "caller" of _ClassB_, si it's up to _ClassA_ to create and inject a _RepositoryB_ into _ClassB_. But There is another dependency between _ClassB_ and _ClassC_. So what should we do here? 
   * *Inject only when it's needed*: _ClassA_ ony injects what _ClassB_ needs, and lets _CLassB_ worry about the further dependency to _ClassC_ 
   * *Inject all at once in a cental point*: _ClassA_ injects the dependences needed by _ClassB_ and *also* the ones needed by _ClassC_. 

As we'll see in the following chapters, these 2 options makes difference in the way we implement our methods.

### Injecting the dependencies only where it's needed

We can implement the injection in the place where it's immediately used. So _ClassA_ injects whatever _ClassB_ needs. But _ClassA_ is not aware of the injections needed by _ClassB_: Those will be managed by _ClassB_. In this approach, these are what the method signatures would look like:

  * If we use the "parameter" approach:
  
  ```
   Value methodA(RepositoryA repoA);
   
   Value methodB(RepositoryB repoB);
   
   Value methodC(RepositoryC repoC);
   
   // example of calling ClassC:
   RepositoryC repoC = new...
   objC.methodC(repoC);
   
   // example of calling ClassB:
   RepositoryB repoB = new ...
   objB.method(repoB);
   
   // example of calling ClassA:
   RepositoryA repoA = new ...
   objA.method(repoA);
   
  ```

  * If we use the "Return Type" approach:

```

   ResultDI<RepositoryA, Value> methodA();
  
   ResultDI<RepositoryB, Value> methodB();
   
   ResultDI<RepositoryC, Value> methodC();
   
   // example of calling ClassC:
   RepositoryC repoC = new...
   objC.methodC().using(repoC);
   
   // example of calling ClassB:
   RepositoryB repoB = new...
     objB.method().using(repoB);
   
   // example of calling ClassA:
   RepositoryA repoA = new...
   objA.method().using(repoA);
   ```

The main advantage of this technique is that the signatures of our methods are simple: apart from the fact that we have moved the State into our method (in the parameter list or in the Return Type), there are quite straightforward.

The main disadvantage is that the injection now takes place in different places all over our project. And this is a real Stopper for our architecture, since keeping the injection in a single point is one of the advantages of the dependency injection.

> *_So we can already say that this option is OUT of the table_*


### Injecting the dependency in a single point

We can implement all the injections in a single place, which is usually the first place in all the Flow Chain (a _main_ in a standalone app, a _Controller_ in a MVC project, etc). Unfortutanlely, this means that we need to carry this dependencies all the way throught the flow, which means that our signatures will get more complicated.


  * If we use the "parameter" approach:
  
  ```
   Value methodA(RepositoryA repoA, RepositoryB repoB, RepositoryC repoC);
   
	Value methodB(RepositoryB repoB, RepositoryC repoC);
	
	Value methodC(RepositoryC repoC);`
	
	// example of calling ClassC:
	RepositoryC repoC = new...
	objC.methodC(repoC);
	
	// example of calling ClassB:
	RepositoryC repoC = new...
	RepositoryB repoB = new...
	objB.methodB(repoB, repoC);
	
	// example of calling ClassA:
	RepositoryA repoA = new...
	RepositoryC repoC = new...
	RepositoryB repoB = new...
	objA.methodA(repoA, repoB, repoC);
	
  ``` 
  
  * If we use the "Return Type" approach:
  
  ```
	ResultDI<RepositoryA, ResultDI<RepositoryB, ResultDI<RepositoryC, Value>>> methodA();
   
   ResultDI<RepositoryB, ResultDI<RepositoryC, Value>> methodB(RepositoryB repoB);
   
   ResultDI<RepositoryC, Value> methodC(RepositoryC repoC);
   
   // example of calling ClassC:
   RepositoryC repoC = new...
   objC.methodC().using(repoC);
   
   // example of calling ClassB:
   RepositoryB repoB = new...
   RepositoryC repoC = new...
   objB.methodB().using(repoB).using(repoC);
   
   // example of calling ClassA:
   RepositoryA repoA = new...
	RepositoryB repoB = new...
	RepositoryC repoC = new...
	objA.methodA().using(repoA).using(repoB).using(repoC);
   
```

you can see that in some cases the method signature can become a nightmare.

### Reader Monad Pattern as an implementation of the "Return Type" option.

If we assume that the methods defined in the previous examples return a String, the following lines are equivalent:

```
RepositoryA repoA = new...
RepositoryB repoB = new...
RepositoryC repoC = new...

// option 1:
System.out.println(objA.methodA(repoA,repoB,repoC);

// option2:
System.out.println(objA.methodA().using(repoA).using(repoB).using(repoC);

```
Even though the option2 looks more complex, it actually makes an important difference: The method needs NO parameter to do it's job, and all the dependencies (which are the _HOW_, not the _WHAT_) are defined at the end.

On the other hand, the first option cam misslead you into thinking that the Repositories are the _WHAT_, when they are not.

> NOTE:
> For me, the WHAT is part of the Contract and is what you need to know in oder to get the info. The HOW, on the other hand, is more part of the implementation, and refers to the way things are done (by using a DB repository, or a Filesystem Repository, for example. Both return the same data, but in different ways)

However, all of this is more of a semantics thing, and it depends to a great extend on the hability of the developers to do it the right way.


Using the "parameter" option is not an option for some SW Architects. Instead, adding the dependency into the Return type seems to have more supporters in the community. As you now already know, the "Return-type" option relies on returning functions instead of values. It's _when_ you use those functions when you inject the dependency. 

The *Reader Monad* Pattern is an implementation of this pattern, which relies in the same principles: defered execution and returning a Function as a Result. 


> More info about the _Reader Monad_:
> 
> [Dependency injection using the Reader Monad in Java 8](https://medium.com/@johnmcclean/dependency-injection-using-the-reader-monad-in-java8-9056d9501c75)

> [A functional approach to dependency injection in Java](https://hackernoon.com/superkleisliisfantasticframeworksareatrocious-a-functional-approach-to-dependency-injection-in-e7bc8c4993fa)


> _NOTE: I'm still not quite sure how to deal with those scenarios where ee have a method with more than 1 dependency. Using nested structures like the ones in the listing above can become very complex very soon. Need to llok into it._
 
 
### My Suggestion: The Composition root Pattern 

> This is my suggested implementation approach, considering that we are NOT using other DI Containers like _Spring_.

We have already agreed that the injection must take place in one single point in our application. A different thing to decide is whether to use a "parameter" or a "Return type" approach.

At the same time, it seems "ugly" to me to make all those changes to our method signatures only to provide them with the dependencies. 

So in this approach, I'll try to make things easier, and at the same time benefit from the Dependency Injection.

#### first: Our methods must be CLEAN.

This means that we should stop adding dependencies in the parameters list, or in the Return Type. So the the two options that we were looking at before:

```
Value method(Dependency dep);
ResultDI<Dependency Value> method();
```
are now turned into this one:

```
Value method();
```

Ok, but now, where is the dependency injected? the answer is: inside the method body. BUT we are not creating the depedenceny directly there, we are not creating any instance directly. Instead, we are going to use another class which we can call the Composition Root.

#### second: Implementing the Composition Root

Let's define a Interface for a "theorical" Composition Root, which purpose is to create instances of a dependency:

```
interface CompositionRoot {
  RepositoryA getRepoA();
  RepositoryB getRepoB();
  RepositoryC getRepoC();
}
```

And now we need an implementation of this interface, like this one which returns valid dependencies to use in a "Production" envirnment 

```
Class ProdCompositionRoot implements CompositionRoot{
  Object getRepositoryA() {return new RepositoryA(); }
  Object getRepositoryB() {return new RepositoryB(); }
  Object getRepositoryC() {return new RepositoryC(); }
}
```

Or this one, which _Mocks_ some of them, to use in a "Test" environment:

```
Class TestCompositionRoot implements CompositionRoot extends ProdCompositionRoot {
	// we mock the RepoC only
	Object getRepositoryC() {return new MockC(); }
}
```

And finally we need a Factory that returns these instances:

```
class CompositionRootProvider {

	public static enum Environment {
		PROD, TEST;
	}
	
	private static Environment ENV = PROD;
	
	private static CompositionRoot compositionRoot;
	
	public CompositionRoot getInstance() {
		if (ENV == PROD) return new MyAppCompositionRoot();
		if (ENV == TEST) return new TestCompositionRoot();
		throw new RuntimeExcetion();
	}
	
	public static void setEnvironment(Environment env) {
		this.env = env;
	}
}
```
 
 And that's it!
 
 Our method now will get its dependency within its body, this way:
 
 ```
 ClassA {
  Value methodA() {
   RepositoryA repoA = CompositionRootProvider.getInstance().getRepoA();
   // We do whatever we have to do with our RepoA, and then return...
   return ...
  }
 }
 ```
 
 And we can set up the _CompositionRootFactory_ at a single point in our application, like in our _main_ method:
 
 ```
 // From the entry-point (main):
  CompositionRootProvider.setEnvironment(PROD);
 ClassA objA = new ClassA();
 objA.method();
 
 ```
 
 In this approach, we have several benefits:
 
   * our methods are free of unnecessary dependencies, which are an implementation thing only.
   * The depenecnies are defined in a single point in our project. We can define it in a separate class like in the example above.
   * Changing from different environment (and changing the dependencies) is a matter of providing a new implementation of the CompositionRoot interface.
 
 
 As a possible drawback that one might think of, is the fact that the Dependency Injection is not really external to our application, but it's part of our code base. There is nothing that stops the developers from messing with the CompositionRoot implementation, or puttin it in the wrong places, or duplicate them in different packages, etc. 
 
 In my opinion, this is not really an issue. When we use _Spring_ as a Deendency Injector Container, we seem to think that the Injection is external to our application, but that's ony because we can NOT see the Code running behind the scenes. And now we can affect the dependency works by using @Configuration Classes in Spring and Spring Boot, so we can no longer say that the Dependency Injection is completely "external" to our application.
 
 It's more of a matter of implementing things right in this case. If we apply the CompositionRoot pattern right, we can have DI and our code clean at the same time.
 
  
 
 
# Final Thoughts

These are my personal opinion and conclusions, which most probably don't match somebody else's, but it's fine, part of the game:
 
 * If we are using Java, implementing pure functions and keeping the dependencies as the _State_ of our Object while we inject them through the constructor using _Spring_ seems to be the best option for me.
 
 * If we have to develop the Dependency Injection by ourselves (in Java or any other language), my preference is NOT to modify the method signatures. To me, a dependency is NOT part of the method contract, but part of the implementtion. For example: if I have a Service thats search a user in a Repository by Name, I only expect the method to get a "userName" parameter. I dont think I should specify also the _Repository_ that service is using, not in the parameter list, nor in the Return Type either. That's part of the _Implementation_ of the method, but it should NOt be part of the contract. That being said, I think that each method should take care of retrieving its dependencies. And implementing all those "dependency creation" code in a single place using the _Composition Root_ patter seems to be the logical option to follow.


 