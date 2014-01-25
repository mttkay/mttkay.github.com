---
layout: post
title: "Monads: Your App As A Function"
date: 2014-01-25 09:15
comments: false
categories:
- functional programming
- rxjava
- monads
---

A paper by Twitter's Marius Eriksen (["Your server as a function"][0]), which introduces
the key concepts behind Finagle is what made me choose the title for this post.
I believe functional programming (FP) will be just as important to mobile
application development in the future as it is for web development today. The
reason is quite simple: it's easier to write resilient code in functional languages,
and resilience is key. Performance might be a feature, but resilience is a
*must*. If the critical paths through your business logic are brittle, then your
app can be as fast as light, but your users will still scoff at it and look
elsewhere for value or entertainment.

I write Android applications, and Java is not a functional language. It's not
even an object-oriented language, at least not in a puristic sense. However,
that doesn't stop us from adopting some of the good practices found in FP to
improve on existing Java code. In this post I'll try to explain how adopting
one of the most fundamental type patterns in FP, monadic types, can dramatically
simplify and improve the robustness of your core application logic.

I have already written about [RxJava and functional reactive programming][3] and how
we make use of it in our mobile applications at SoundCloud. 
I hope it served as a good introduction
into using that library specifically, and how expressing expensive operations
through Observables makes your code more resilient to failure and easier to
compose. However, there's a reason why Observables are so
universally useful: they're monads. This post is my own attempt at explaining
monads, why they're so valuable, and why you should consider using them.

Before you keep reading--or, heavens forbid, consider dropping out here!--let me say that none
of the following pragraphs will assume you have experience with FP or any functional
language for that matter. I will use Java for all examples, so that you have
something familiar to work with if you're a Java (or Android) developer already.

# Say Monad one more time...
It's almost a joke these days. People hate it when FP folks start talking about
monads. People hate it, because they have a vague idea at best of what a monad
is, and it makes you feel like an idiot. No one wants to be an idiot! Let me
tell you: you're not an idiot, and monads are not difficult to understand. It's
just surprisingly difficult to explain them.

I'm not a mathematician. I don't know category theory. But I believe I have
understood monads to a degree that I can make effective use of them in the code
I write day in and day out, and that I can even write my own monadic types.
Here's another piece of good news: if you understand monads, you understand most
of the underpinnings of functional languages. You'll quickly find when jumping
from one FPL to the next, monads will follow you around. It's a bit like
understanding classes in object-oriented languages. If all you've ever known is
procedures and value types, then classes may seem odd at first. But once you
understand classes, it doesn't matter which OOPL you use, the concepts remain
the same.

So what's a monad? I think monads are best explained (and appreciated!) by 
realizing in what poor situation you as an imperative programmer actually are.
So I'll start by showing you a piece of code that I bet you've written
yourself in some way shape or form at some point in time, 
and then making you reflect on why your code sucks. No offense by the way, my code sucks too. 
But that's the great thing about being a developer,
right? We strive to make code suck a little less every single day.

Let's look at the example.

# A piece of code you've written before
If you're an app developer, chances are you connect to some service API to
download JSON or XML that describes your business objects. At SoundCloud,
everything evolves around tracks, so we download track metadata a lot. If
you're doing it right, then you're also caching this data somewhere. It doesn't
really matter where or how, it could be in a database or just using flat files.
Here's a very typical of way of doing this in Android using the AsyncTask class:

{% codeblock lang:java %}
class FetchJsonObject extends AsyncTask<String, Void, JsonObject> {

  protected JsonObject doInBackground(String... args) {
    final String url = args[0];
    String json = serviceApi.get(url).readString();
    cache.persist(url, json);
    return JsonObject.parse(json);
  }

}
{% endcodeblock %}

Don't worry too much about what types JsonObject or serviceApi are here. This is just pseudo
code serving to get my point across. It should still be easy to see that we're trying
to achieve the following:

1. Download a JSON document from a given URL
1. Cache it to disk using the URL as the cache key
1. Parse it into an in-memory representation

Instead of actually pointing out what the problem with all this is, let's turn
this into a short Q&A. Have a look at the following questions. What are *your* answers
to each of them?

#### Q: Every single line here can throw an exception. Where is it handled?
A: Simple, you wrap everything in a try/catch block. Fair enough. Then what?
How do you propagate the exception to the caller? Recall that this job is running
on a background thread, so there might be visibility issues. Moreover, how do
you signal the error? You have to return something from `doInBackground`. Thinking
about returning null? You might want to listen to what [Tony Hoare has to say about
null references][1] (he invented them by the way.)

#### Q: If the API request fails, what do we return?
A: Easy, we return null! Sorry, but [Tony Hoare says NO][1], so let's put our
foot down on that one okay?

#### Q: If the API request succeeds but caching fails, do we throw out the result?
A: Uhm, maybe? We haven't really thought about how to propagate the data in this
series of steps. Aha! There's our first clue about what a monad is: transporting
data as a series of steps. I'll pick this up again later.

#### Q: Should caching to local storage happen on the same thread as sending API requests?
A: Probably not. Because this could mean that API requests block local storage
I/O from happening in case they take longer than expected, right? It almost looks
as if caching to local storage should happen in its own task. Or maybe it
has something to do with propagating data as a series of steps... (This is where
you should picture me waving a flag in your face that says monad on it.)

#### Q: If I want to just make an API request/just cache to local storage, do I write a new AsyncTask for each?
A: I suppose so. If we did that, however, how would be combine them to arrive
at the definition above, which performs both steps in succession and pipes data
from one task to the next? I smell sulfur, we might be well on our way to [callback hell][2].

I hope the picture begins taking shape. It appears there are a number of related
problems here, most of them having to do with *processing data as a series of 
potentially asynchronous steps where failure in each step is anticipated.*

Let's finally look at what monads are and how they solve this for us.

# Monads explained
We'll get a little more concrete now and jump straight into the definition of
what monads are and what has to hold true for a monad to actually be a monad. 
I then show how a simple monadic type could look like in Java.

Let's first look at some of the existing definitions that attempt to put
monads in a single sentence. Erik Meijer, the man behind the [Reactive Extensions][4]
and my personal hero for wearing a SoundCloud t-shirt on stage at GOTO Berlin,
has this to say about monads:

> Monads are return types that guide you through the happy path.

This might be my favorite definition, because it catches the gist of what monads are
all about. However, it's still a bit vague and doesn't really help in understanding
what the structure of a monad is. Martin Odersky, EPFL fame and inventor of the
Scala programming language looks at it this way:

> Monads are parametric types with two operations flatMap and unit that obey some algebraic laws.

So this is rather the opposite: this definition doesn't really tell us what monads
are good for, but it contains some important clues about their structure, i.e. they
are types, parameterized over another type, and they consist of just two operations.
I told you monads were simple!

Both these definitions I took almost verbatim from the reactive programming course
on Coursera, which I highly recommend. Let's have a look at what Wikipedia has to say:

> Monads are structures that represent computations defined as sequences of steps.

Sounds familiar? I told you I'd come back to the whole sequence of steps thing.
Finally, here's my own attempt at putting monads in a sentence, and it's the definition
I will use throughout the rest of this article:

> Monads are chainable container types that trap values or computations and allow them to be transformed in confinement.

The key take away from this definition are the "three Cs":
**containment, chainability, and confinement.**

I will now explain how monads enable these properties for arbitrary data or
computations and why that's super awesome.

## Monads are types
We just learned that monads are parametric types that define just two operations, `unit`
(also called return) and `flatMap` (also called bind or mapMany). I will
show you shortly what these methods do and what a full definition of a monad
looks like in Java, but just to put your worries to rest a little: if you've used
Scala or RxJava before, then you've already seen and used
monads. All lists in Scala are monads with the `List` constructor method as 
unit and a flatMap method to transform them. Observables in RxJava are monads
with `Observable.from` as unit and mapMany to transform them
(mapMany in RxJava is actually aliased to flatMap, so you can use either one.)

That said, let's have a look at the structure of a monadic type in Java:

{% codeblock lang:java %}
public class Monad<T> {
  private T value;

  private Monad(T value) {
    this.value = value;
  }
  
  public static Monad<T> unit(T value) { 
    return new Monad(value); 
  }

  public <R> Monad<R> flatMap(Func1<T, Monad<R>> func) {
    return func.call(this.value);
  }

  public T get() {
    return value;
  }
}
{% endcodeblock %}

I've added a third method here, `get`, but it's hardly worth talking about a
getter function, so let's skip it as part of the discussion and turn straight
to `unit` and `flatMap`.

## The 1st C: unit enables containment
There's an obvious take away from the snippet above: a monad is defined over a type `T` which
it contains values of. Containment here is enabled by the `unit` function: it takes a `T`
and traps it in the monad by creating a new instance of the monad with that value
passed into it. At this point I should mention that `T` can be anything, including
collection types like lists. Remember RxJava and observables? An Observable is
a monad defined over collections of values. Let's keep things simple though
and let's create a monad of integers:

{% codeblock lang:java %}
Monad<Integer> intMonad = Monad.unit(2);
{% endcodeblock %}

So we stuff the number 2 in our monad. Cool, but not so useful thus far. 
What makes it useful is the flatMap method, so let's turn to flatMap. 

## The 2nd C: flatMap enables chainability

I admit this one might look a bit more puzzling, but it's
actually pretty straight forward once you look past the awkward syntax. The `flatMap`
function itself is defined over a new type variable, `R`, and it returns a new monad
of that type `Monad<R>`. It does so, however, not just by trapping the value in it like
unit does, but by applying a function to the current value, a function which
knows how to turn `T`s into monads of `R`. (I borrowed the `Func1` type from
RxJava here: it means it's a function object that takes 1 argument of type `T`
and returns something of type `Monad<R>`.)

Let this sink in for a second, since this is perhaps the most important aspect
of monads. It's important because it allows us to chain monads together using
transformations of the values they contain. I can take a monad containing, say,
an integer, and flatMap it using a function which takes this integer, transforms
it (say, by taking the square root of that number) and sticking it in a new monad. The last part
is critical, since it means we can do this forever and ever, because the return
value will be a monad again with a flatMap function which can again take a
function which returns another monad which has... you get the idea.

Let's take the square root example using the monad we just created:

{% codeblock lang:java %}
double result = intMonad.flatMap(new Func1<Integer, Monad<Double>>() {
    public Monad<Double> call(Integer input) {
      return Monad.unit(Math.sqrt(input));
    }
  }
}).get();
{% endcodeblock %}

Now that looks more useful! What we've done here is take our initial integer
monad and transformed it using flatMap to obtain a new monad of type `Monad<Double>`
that contains the square root of the initial value. This is fundamentally
different from applying the `sqrt` function directly to some input, since
there's no flatMap method defined on `double` that you could use to apply
further transformations, so you'd effectivly lose the property of chainability.

This also enables entirely new perspectives in terms of code structure and reuse:
since the monad structure never changes, your business logic is
entirely expressed in terms of functions, which transform values step wise,
are defined and tested in isolation, and composed together to form new
pieces of functionality. It's like Legos, but using functions.

## The 3rd C: To be continued...

If you paid attention at the beginning, you might have noticed that the "3rd C", namely
confinement is still missing. This is because is has to do 
with the algebraic laws a monad has to obey. Frankly, I haven't been completely honest with you.
Our sample type follows a monadic structure, but purists will say it's not
actually a monad. There's a more subtle aspect of flatMap than just being able
to chain things together, and it has all to do with making monads resilient
and not allowing things to leak out of the monad.

Why that is the case, how it has to do with the three monad laws, and how we
can piece everything together to turn our initial AsyncTask example into
something fundamentally better using monads is what I will cover in part 2 of this article.


[0]: http://monkey.org/~marius/funsrv.pdf
[1]: http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare
[2]: http://ianbishop.github.io/blog/2013/01/13/escape-from-callback-hell/
[3]: http://mttkay.github.io/blog/2013/08/25/functional-reactive-programming-on-android-with-rxjava/
[4]: https://rx.codeplex.com/