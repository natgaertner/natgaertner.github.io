---
layout: post
title:  "Semantic Maintenance"
date:   2019-03-04 20:13:45 +0000
categories: programming 
---
One of the great struggles of programming is to say what one means. We experience this problem at the language level consistently, as anyone can attest who has written:
{% highlight java %}
return list.get(list.size() - 1);
{% endhighlight %}

When they wished they could write:
{% highlight java %}
return list.last();
{% endhighlight %}

(And if you have never screwed up the index when trying to get the last thing in a list, well I guess you are just more #blessed than us hoi polloi).

Of course, sometimes this comes down to a matter of the language matching one's intuitive modeling. I obviously am prone to thinking I am correct that this Python function
{% highlight python %}
def map(list_param, func_param)
    return [func_param(_) for _ in list_param]
{% endhighlight %}

is more comprehensible than this Ruby equivalent
{% highlight ruby %}
def map list_param
    return_list = []
    list_param.each { |elem| yield(elem) >> return_list}
    return_list
end
{% endhighlight %}

but I learned Python before I ever saw Ruby, so I have to also admit that I have been pre-biased to model the problem a particular way.

Languages also can improve over time. Java 8 gave us ```Optional``` and replaced this:
{% highlight java %}
final Pizza pizza;
if (oliveJar != null) {
    pizza = new Pizza(oliveJar.getContents());
} else {
    pizza = new Pizza(new BoringToppings());
}
{% endhighlight %}

with this:
{% highlight java %}
final Pizza pizza = new Pizza(Optional.ofNullable(oliveJar)
    .map(OliveJar::getContents)
    .orElseGet(BoringToppings::new));
{% endhighlight %}

which is inarguably (do not try to disagree I said inarguably) much better.

# Semantic Maintenance for You and Me
Most of us are not language maintainers in the formal sense, but often we do preside over a set of semantics. The data models, configurations, and class hierarchies of our applications are tools not just to achieve specific ends, but also to express concepts. Often one of the first tasks of designing a new application is to define its data model (If you are lucky enough to run a stateless application, go find whoever is implicitly serving as your data store and thank them). The connection between the initial data model definition and the modeled concepts underlying the application functionality is usually fairly clear. Users have constituent components such as names and passwords and profile data, there is usually a master object for whatever "thing" the application is helping users manage, etc. Relationships between objects sometimes get their own models as well, and often this is where expressing meaning gets tricky. Relationships are often contextual, as anyone who has had a drinking buddy run into them out with work friends knows. The problem with contexts is that new ones keep showing up. Relationships may only be relevant in some contexts, and objects may perform entirely different roles in new use cases. I'll give a particularly insidious example from my Real Life.

Let's say you have a type `Foo` that serves as a node in a graph structure. Aside from the usual node property of having incoming and outgoing edges, Foo has two properties: a color, and whether it is a source (only outgoing edges), a sink (only incoming edges), or neither (incoming and outgoing edges).
{% highlight java %}
public class Foo extends Node {
    private Color nodeColor;
    private GraphRole role;
}
{% endhighlight %}

```Color``` and ```GraphRole``` are enums with a fixed discrete set of possible values:
{% highlight java %}
public enum Color {
    BLUE,
    RED,
    PUCE;
}

public enum GraphRole {
    SOURCE,
    SINK,
    MIDDLE;
}
{% endhighlight %}

Immediately we are allowing some confusion here since GraphRole could be descriptive (what kind of edges does it have?) or normative (what kind of edges is it *allowed* to have?) Let's clarify: it's normative. So when constructing a graph of Foos, GraphRole is declared when a node is created, and adding an incoming edge to a source or an outgoing edge to a sink is an illegal operation. We're pretty happy with this, we've set up some nice modeling of the properties people need to dynamically create graphs of Foo nodes.

Over time, as our tool's usage grows, we get requests from users. "It would be great if I could frobulate nodes" say users. We agree, that would be great, so we implement it. Now, as everyone knows, you CANNOT frobulate a Puce Sink, so we write some code like this:
{% highlight java %}
public void frobulate(final Foo node) {
    if (node.getNodeColor() == Color.PUCE && node.getGraphRole() == GraphRole.SINK) {
        throw new IllegalArgumentException(
            "Cannot frobulate a puce sink. You know better!");
    }
    // start frobulation
    ...
}
{% endhighlight %}

Makes perfect sense! This works great for years, and we employ this type of filtering for all sorts of functions and roles that only apply to certain kinds of nodes: a greezelax must be a blue source, a red middle means a resonance capability of 5 but a red sink means 7 and any other color is 3, etc. Finally, the times are catching up with us, and a new color of node becomes available: Cerulean. Now we have a problem. Can you frobulate a Cerulean Sink? Could a greezelax be a Cerulean middle? What's the resonance capability of Cerulean nodes? Now we have to find everywhere in the code we made these kind of filtered judgments and fix them. Should we just add some cases to our conditional statements to handle the new color? Maybe, but when Burnt Umber comes online next year we're going to regret it. 

The problem here is that we've overloaded the semantics of our data. We've introduced all these concepts: frobulating, greezelaxes, resonance capability that are nowhere in our model. The solution is to expand our semantics. For frobulating and resonance capability the approach is fairly obvious:
{% highlight java %}
public class Foo extends Node {
    private Color nodeColor;
    private GraphRole role;
    private boolean frobulatable;
    private int resonanceCapability;
}
{% endhighlight %}

Now we can set these properties explicitly rather than deriving them. But what about greezelax? Say we have a function that looks like this:
{% highlight java %}
public Foo createFromGreezelax(final Greezelax greezelax) {
    return Foo.builder()
        .withNodeColor(Color.BLUE)
        .withGraphRole(GraphRole.SOURCE)
        .withOutgoingEdges(greezelax.getEdges())
        .build();
}
{% endhighlight %}

The assumption is built into the code in an unfortunately stubborn way. Does the greezelax provide any data to tell us whether it is a Blue Source or a Cerulean middle? Maybe not! Users have been used to passing in a greezelax and happily assuming the outcome this whole time, but now we have this new class of users with this different kind of greezelax. The old users have no idea about these new people, and would definitely be annoyed if all their code where they don't specify their greezelax type started failing all of a sudden. We are sadly now in the difficult world of attempting to deprecate old behavior, which is an entirely different topic for someone smarter to write about, but a possible solution is to introduce defaults:
{% highlight java %}
public Foo createFromGreezelax(final GreezeLax greezelax) {
    return Optional.ofNullable(greezelax.getRodeoStyle())
        .map(rodeoStyle -> this.createFromGreezelax(greezelax, rodeoStyle))
        .orElseGet(() -> Foo.builder()
            .withNodeColor(Color.BLUE)
            .withGraphRole(GraphRole.SOURCE)
            .withOutgoingEdges(greezelax.getEdges())
            .build());
}

public Foo createFromGreezelax(
        final Greezelax greezelax,
        final RodeoStyle rodeoStyle) {
    if (rodeoStyle == RodeoStyle.GRAVY) {
        return Foo.builder()
            .withNodeColor(Color.CERULEAN)
            .withGraphRole(GraphRole.MIDDLE)
            .withOutgoingEdges(greezelax.getEdges())
            .build());
    } else {
        return Foo.builder()
            .withNodeColor(Color.BLUE)
            .withGraphRole(GraphRole.SOURCE)
            .withOutgoingEdges(greezelax.getEdges())
            .build());
    }
}
{% endhighlight %}

Whew, that's that handled! Except, what if some new rodeoStyle specification happens, and it wants some totally different Color and GraphRole? Oh geeze these rodeo people are the worst and they are ruining our perfect model. Maybe something like this instead:
{% highlight java %}
public Foo createFromGreezelax(
        final Greezelax greezelax,
        final RodeoStyle rodeoStyle) {
    return Foo.builder()
        .withNodeColor(rodeoStyle.getColor())
        .withGraphRole(rodeoStyle.getGraphRole())
        .withOutgoingEdges(greezelax.getEdges())
        .build());
}
{% endhighlight %}

What we've done is further refined the semantic model of rodeo styles so they can express their colors and roles, rather than have us roughly interpret them. Semantic maintenance is something we can do for ourselves as application maintainers, but eventually it is driven by use cases and can reach all the way up to supplying users with more direct ways to say what they mean. If the RodeoStyle Standards Conclave wants to introduce new styles with new colors and roles, we have now enabled them to do so without directly asking us to change our handling. 

As a final note, I've talked here about doing semantic maintenance in response to new use cases, but often what happens is that our original semantics were insufficient for all the situations that are *already* expressable. The perfect data model is rare, even for a static set of use cases. If it is ever achieved, it is through iteration that somehow exceeds the pace of user evolution. Good luck?
