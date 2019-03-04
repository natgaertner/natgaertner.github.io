---
layout: post
title:  "Types Ahoy (Declare More Types Please)"
date:   2019-03-04 19:49:45 +0000
categories: java
---
Consider the following line of Java code:
{% highlight java %}
final ListMultimap<Foo, SetMultimap<Bar, Fizz>> barsWithFizzesBelongingToFoos =
    ArrayListMultimap.create();
{% endhighlight %}
```ListMultimap``` and ```SetMultimap``` are [Google Guava][guava_link] collections classes that
represent, repsectively, a ```Map``` whose values are ```List```'s and a ```Map``` whose values are ```Set```'s. In plain Java this would look something like:
{% highlight java %}
final Map<Foo, List<Map<Bar, Set<Fizz>>>> barsWithFizzesBeloingingToFoos =
    new HashMap<>();
{% endhighlight %}
It\'s rare for a single line of code to mean much of anything, but an immediate problem is visible here: This represents a complex, nested relationship among 3 object types: ```Foo```, ```Bar```, and ```Fizz```, but we have no idea what the relationship means or where the values of Foo, Bar, and Fizz will come from. The instances have no names, merely their designations as keys or values in their respective maps. Perhaps our knowledge of the codebase this came from can help us understand why these objects are arranged this way, but it seems unlikely that this represents a relationship modeled in the object structure of the code. Were that the case, we would have used that modeled relationship to arrange the objects here. This gets even more confusing if we replace Foo, Bar, and Fizz with more primitive types:
{% highlight java %}
final ListMultimap<Long, SetMultiMap<Long, String>> barsWithFizzesBelongingToFoos =
    ArrayListMultimap.create();
{% endhighlight %}
We can maybe guess from the variable name what these primitive types are meant to be. They are probably some meaningful fields of the Foo, Bar, and Fizz classes, but even if this were discernable from the limited data we have here, it\'s a lot of extra work.

Criticism is easy, but it\'s also easy to see how this nested collection type came to be used. As noted above, this probably represents a relationship that is not modeled natively by the objects, and there\'s a fair chance these objects are only arranged this way in this one instance. Cluttering up the Foo, Bar, and Fizz objects with this relationship might not make sense for the clarity of the code in those classes even if it would help here. Those objects also likely have a more natural relationship that is modeled, and adding another would confound a reader trying to figure out the core relationship among them. It would make an arrangement only relevant in this single context visible in all contexts. The worst thing we can do as programmers is make someone read more of our code when they otherwise wouldn\'t have to.

In addition to these reasons for not piling another relationship into these core classes, the natural flow of programmer-like thinking that leads to arranging them in thes nested collections is also easy to imagine. MultiMaps usually represent some kind of \"group by\" operation. Foos can have a bunch of Bars in them, and each Bar can be in one Foo at a time, so given a collection of Bars, let\'s arrange them according to what Foo they are in. This is going to be useful later when we have to run an operation that expects all the Bars in a given Foo. Additionally, each Bar can be assigned a set of Fizzes, and we\'ve got this other collection of Fizzes and what Bar the\'ve been assigned to. We\'ve created this nice look-up table so that when we find out what Fizzes are assigned to a Bar, we can ask the Bar \"hey what Foo do you belong to?\" take the result, look it up in the map (hash lookups are fast so we\'re being very efficient here), then go add a map from that Bar to the set of assigned Fizzes to the given Foo\'s list. (If you\'re wondering why the values assigned to Foos are Lists of Maps rather than just Maps, all I can say is that I wondered that too and couldn\'t find the reason, but this was pulled from real code and people make mistakes.)

So I\'m ok believing we got to this place perfectly logically (except maybe the Lists of Maps thing), but that doesn\'t help us figure it out, except maybe to give us a bit more empathy for the author and resist the urge, if we know where they work, to go ask angry questions about why they have done this. (It\'s good to resist this urge, since after all the author may work at our very desk.) What can we do to fix it? We talked about the issues of modeling these relationships on the classes themselves, but maybe we can model the relationship more clearly locally! Since this is Java, we\'ll allow ourselves the luxury of long type names, in keeping with tradition. Something close in meaning to our original variable name might be a good place to start:
{% highlight java %}
    public class FizzedBarsInFoo {
      final Foo containingFoo;
      final List<SetMultimap<Bar, Fizz>> fizzedBars;
    }
{% endhighlight %}
This doesn\'t seem like much, it\'s just a tuple representing the entries in the ListMultimap we had before, but it gives us a chance to say more about what we\'re doing, because we get a class name and field names to work with. This becomes more obviously helpful when the member objects are primitives:
{% highlight java %}
    public class FizzedBarsInFoo {
      final Long containingFooId;
      final List<SetMultimap<Long, String>> fizzRepresentationsByBarId;
    }
{% endhighlight %}
Of course, we still have this unpleasant complexity in the second field. We can use the technique recursively to deal with this:
{% highlight java %}
    public class FizzedBar {
      final Long barId;
      final List<String> fizzRepresentations;
    }

    public class FizzedBarsInFoo {
      final Long containingFooId;
      final List<FizzedBar> fizzedBars;
    }
{% endhighlight %}
We thankfully get to skip the unpleasantly verbose ```fizzRepresentationsByBarId``` because we have delegated those details to the ```FizzedBar``` class. We\'ve preserved the List type of the fizzedBars field, consistent with the original capacity to have multiple Maps from the same Bars to the same Sets of Fizzes under the same Foo. Maybe there is some reason for this that makes sense to the original author. Creating more classes can help make explicit what we otherwise might have to rely on implicitly. Maybe the existence of duplicate entries indicated a need for special processing in the code that eventually reads this data structure. There are possibilities to model this explicitly:
{% highlight java %}
    public class FizzedBarsInFoo {
      final Long containingFooId;
      final MultiFizzedBars fizzedBars;
    }

    public class MultiFizzedBars {
      final Set<FizzedBar> uniqueFizzedBars;
      final Set<FizzedBar> duplicatedFizzedBars;
    }
{% endhighlight %}
This is all much tidier, and gives us a chance to hide some of the inner workings of each sub-relationship from the original calling code and anyone reading it. But maybe it is a lot of extra structure just to handle one little mapping that we only need in one place. What counts as a cluttering number of classes is perhaps an issue of taste and style, but I often find myself feeling that the code I see indirectly and the code written in my immediate view is too timid about adding more classes. If they truly are required only in one place, they can even be private inner classes and never be known to anyone not opening up that particular parent class file. If they are required in more than one place, this is even more of an argument to define the relationships involved somewhere concrete. In the example code I pulled this from, the nested collection data structure had for a long time been used only in one place, but was about to be passed out and referenced as a parameter in an asynchronously run job elsewhere. If the meanings of the members of the structure were obscure in the code that created it, they were doubly so in code that only consumed it. Consider interpreting something like:
{% highlight java %}
    public void reticulateSplines(
            final ListMultimap<Long, SetMultimap<Long, String>> foosToProcess) {
        ...
{% endhighlight %}
This is no fun!

High level programming languages work by expressing analogies about the real things (or closer to real things) happening when they run. These analogies represent the real happenings in a way that is hopefully understandable by the programmers and interpreted deterministically by the compiler or language interpreter. The latter is guaranteed (or close enough to guaranteed for our purposes) by the semantic rules of the language itself. The former is a constant struggle. Human languages also allow us to create analogies to real happenings (or, once again, closer to real), and their power lies in the extent to which they are expressive. The tools of human language expression are well beyond the study of computer programmers, but we can flatter ourselves that we can name a few: vocabulary, tone, grammar. Our tools for communicating our meaning to other human readers of our computer code are much more limited, but we have the great expressive technique of declaring new vocabulary elements: types, variables, functions, etc. We get to invent new phrases and declare them to have a precise meaning that must be understood by the reader as they continue through the text (French philosophers also love to do this). If creating new types feels difficult in a codebase, it hinders one of the few expressive tools we have.
[guava_link]: https://github.com/google/guava
