---
layout: post
title:  "Java Generics Intuition"
date:   2019-03-04 20:33:45 +0000
categories: java
---
Almost all Java developers have encountered a mysterious error when working with Java generics. Sometimes it\'s just not remembering where to parameterize generic functions (right before the return type). Sometimes it\'s the surprising failures of Java\'s type interpolation in stream pipelines. Personally, gaining a working intuition for when \"extends\" and \"super\" will work and when they are appropriate has been one of my hardest tasks as a Java developer. Here I\'ll discuss how some of the simpler intuitions are modeled in my head, and perhaps that will be useful if you are encountering similar challenges.

Examine this code:
{% highlight java %}
public Optional<Foo> getFoos() {
    return Stream.of(new FooImpl(), new FooImpl()).findFirst();
}
{% endhighlight %}
At first glance (certainly at many of my first glances) this code seems fine. `Foo` is an interface, `FooImpl` is an implementation of `Foo`. `Stream.findFirst` returns an `Optional` containing the first element of the stream, which contains a `FooImpl`, which *is* a `Foo`. Perfect!
{% highlight sh %}
src/nat/demo/generics/GenericsDemo.java:10: error: incompatible types: Optional<FooImpl> cannot be converted to Optional<Foo>
        return Stream.of(new FooImpl(), new FooImpl()).findFirst();
{% endhighlight %}
Um, what do you MEAN `Optional<FooImpl>` cannot be converted to `Optional<Foo>`???? This is preposterous. The obvious falsehood before us sneers in its outrageous self confidence.

Unfortunately, it is correct, and the difference (at least in my mind) has to do with the distinction between referential labels and actual instances. The `Optional<Foo>` used as the return value of `getFoos` is not an instance of anything. It is a label that tells us what kind of object may be returned from `getFoos`. It is exactly equivalent to the type declaration for a variable that tells us what kind of objects may be assigned to that variable. Code such as
{% highlight java %}
private final List<Foo> foos = new ArrayList<FooImpl>();
{% endhighlight %}
Gives us an equivalent error:
{% highlight sh %}
src/nat/demo/generics/GenericsDemo.java:10: error: incompatible types: ArrayList<FooImpl> cannot be converted to List<Foo>
    private final List<Foo> foos = new ArrayList<FooImpl>();
{% endhighlight %}
(note the error has nothing to do with the difference between `ArrayList` and `List`. It would occur exactly the same if the declared type of `foos` were `ArrayList<Foo>`.)

In both the case of returning a value from `getFoos()` and assigning `foos` we have an actual instance in hand: an instance of type `Optional<FooImpl>` and an instance of type `ArrayList<FooImpl>`. These instances are assigned to the return value of the function or the variable we are setting, and from then on, when either of those values is referred to, we will get these instances. This is where bad things can start happening. Consider the humble `add` function of the `List` interface. In its simplest form, `List<E>.add` takes one parameter of type `E`. \"yes yes yes and in this case Foo is E and FooImpl is a Foo so it is also an E so everything is fine\" we want to say. Java is wasting our time AS USUAL. We can totally call `foos.add(new FooImpl());` and it will work great. But interfaces are of course made to be implemented, and an interloper appears: `BetterFooImpl`. Of course some jerk thinks they can do better than us. Nothing wrong with the original FooImpl but ok friend you do you we have better more lofty concerns to get on to, genius programmers that we are. So, if you can believe it, this DOOFUS calls `foos.add(new BetterFooImpl());` Ok, that\'s offensive, that was our list, and they shouldn\'t be putting their stuff there, but ok we can\'t stop them. But let\'s imagine that the absurd (bad, false, stupid) Java error above didn\'t happen and we had successfully assigned `foos` a value of type `ArrayList<FooImpl>`. What would `foos.add(new BetterFooImpl());` do now? Well if this somehow managed to happen at runtime (ok I guess via reflection or something? I am too lazy to write that out) who knows what nonsense would occur. We can kind of see at compile time if we try to add an instance of `BetterFooImpl` to a variable of type `List<FooImpl>`:
{% highlight sh %}
src/nat/demo/generics/GenericsDemo.java:18: error: no suitable method found for add(BetterFooImpl)
        foos.add(new BetterFooImpl());
            ^
    method Collection.add(FooImpl) is not applicable
      (argument mismatch; BetterFooImpl cannot be converted to FooImpl)
    method List.add(FooImpl) is not applicable
      (argument mismatch; BetterFooImpl cannot be converted to FooImpl)
{% endhighlight %}
You can\'t add `BetterFooImpl` instances to a list that can only contain `FooImpl` instances! The former is *not* an instance of the latter! (darn right I would never be associated with the moron who wrote `BetterFooImpl`.)

To make it even more clear, think of what would happen if we could add different types to a list, then re-assign the instance to another variable:
{% highlight java %}
final List<FooImpl> definitelyFooImpls = new ArrayList<FooImpl>();
final List<Foo> definitelyFoos = definitelyFooImpls;
definitelyFoos.add(new BetterFooImpl());
final List<FooImpl> anotherFooImplList = new ArrayList<FooImpl>();
anotherFooImplList.addAll(definitelyFooImpls);
{% endhighlight %}
This gets us in deep trouble! Remember that when we add something to `definitelyFoos` the instance that is getting modified is the referenced instance `definitelyFooImpls`! We have added a `BetterFooImpl` to our `definitelyFooImpls` list and what\'s worse, we have handed `anotherFooImplList` those values as well!

Note that an instance that *is* parameterized with `Foo` is perfectly fine accepting any implemenation of `Foo` as a member. It will never tell its users that it contains anything more specific than `Foo`!

So we get it, though we may not like it: A class parameterized with an implementation does *not* extend the same class parameterized with a superclass of that implementation. This has to do with what the class can \"accept\" to its methods. A class which *only* returns the parameterized object from its methods may intuitively be a candidate for having this kind of extension work, but once a class is parameterized, it may use that parameter freely in whatever ways it likes.

But what about designing interfaces?
====================================

A common place this restriction frustrates us is in interface design. Consider the following interface:
{% highlight java %}
package nat.demo.generics;

import java.util.List;

public interface SoobertFoo {
    List<Foo> getAllScooberts();
}
{% endhighlight %}
And this implementation:
{% highlight java %}
package nat.demo.generics;

import java.util.Arrays;
import java.util.List;

public class ScoobertFooImpl implements ScoobertFoo {
    @Override
    public List<FooImpl> getAllScooberts() {
        return Arrays.asList(new FooImpl());
    }   
}
{% endhighlight %}
We *know* that `ScoobertFooImpl` always returns a list of `FooImpl` so we say so. We are so helpful!
{% highlight sh %}
src/nat/demo/generics/ScoobertFooImpl.java:8: error: getAllScooberts() in ScoobertFooImpl cannot implement getAllScooberts() in ScoobertFoo
    public List<FooImpl> getAllScooberts() {
                         ^
  return type List<FooImpl> is not compatible with List<Foo>

{% endhighlight %}
WE TRIED TO BE HELPFUL BUT NOOOOO. Of course, the return type doesn\'t match. We\'d curse the interface author, but it is us. Thankfully, there is a modification we can make to the interface to make this work:
{% highlight java %}
package nat.demo.generics;

import java.util.List;

public interface ScoobertFoo {
    List<? extends Foo> getAllScooberts();
}
{% endhighlight %}
Now our implementation compiles happily. By assigning the type label `List<? extends Foo>` we are saying that the value could be a List of *any* type that extends `Foo`. Do NOT confuse this with saying it is a list that can *accept* any type that extends `Foo`! That would be `List<Foo>`. An implication of this is that we have given up entirely on *any* ability to use functions of `List` that take arguments matching the parameter type. What instance could satisfy `? extends Foo`? We might think \"anything that extends `Foo`, duh\", but that is not what is being asked for here. A parameter of `? extends Foo` says that anything satisfying that type must *extend* `? extends Foo`. Not a particular instance of a type that extends `Foo`, but *all* the possible types that extend `Foo`. It essential says \"to be a valid argument to `List<? extends Foo>.add` you must be a ? for all possible ?s that extend `Foo`.\" Of course everyone can write their own extension, so the possibilities are literally boundless. No single instance can satisfy this endless collection of required superclasses.

For purposes of allowing extensions of interface methods, parameterizing return types with `? extends` is handy, but it tells us no further information about the types in the list as the *consumers* of that function result. We are stuck treating them as `Foo` unless we have somehow gotten our hands on the implementation itself, in which case we know from its handy extension of the interface contract that we can get `FooImpl`. Once we have lost that information (say by passing to a function which takes the interface as a parameter rather than the implementation), there is no getting it back.

Some future time I\'ll talk about the other side of the coin: `? super` and its (perhaps surpring) utility.
