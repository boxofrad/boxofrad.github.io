---
layout: post
title: Nameless parameters in RubyMotion
date: 2015-07-18
---

<p class="intro">
  <span class="dropcap">W</span>hile working on an iOS app for a client I came across an interesting language-level bug (or missing feature?) in RubyMotion.
</p>

The app has a fancy animation which needed a method on 
[CAMediaTimingFunction](https://developer.apple.com/library//ios/documentation/Cocoa/Reference/CAMediaTimingFunction_class/index.html) called 
[functionWithControlPoints](https://developer.apple.com/library//ios/documentation/Cocoa/Reference/CAMediaTimingFunction_class/index.html#//apple_ref/occ/clm/CAMediaTimingFunction/functionWithControlPoints::::). 
It turns out that this is one of the places you'll see a seldom used feature of
Objective-C: Nameless Parameters.

A function with Nameless Parameters can be defined like so:

{% highlight objc %}
+ (void) sayHello:(NSString *)firstName:(NSString *)lastName;
{% endhighlight %}

&hellip; and instead of the usual *key: value* style invocation you can simply
omit the keys:

{% highlight objc %}
[Greeter sayHello:"Daniel" :"Upton"];
{% endhighlight %}

It's a neat trick, but one that Apple [actively discourages](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingMethods.html#//apple_ref/doc/uid/20001282-1001751-BCIJHEDH)
and it seems the RubyMotion team aren't too keen either:

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">Nameless
Objective-C parameters are the most hideous thing I have ever had the
displeasure to work with. Also completely unnecessary.</p>&mdash; Eloy Durán
(@alloy) <a href="https://twitter.com/alloy/status/449500090416500736">March 28,
2014</a></blockquote>

Fortunately they crop up in few places, but unfortunately for us CAMediaTimingFunction is one of them and RubyMotion doesn't support the syntax (check out the discussion in [RM-123](http://hipbyte.myjetbrains.com/youtrack/issue/RM-123)).

With a little <del>magic</del> meta-programming you can poke still poke at the method, here's
what I ended up doing:

{% highlight ruby %}
# [CAMediaTimingFunction functionWithControlPoints:0.4, :0.0, :0.2, :1.0];
def media_timing_function
  signature = NSMethodSignature.signatureWithObjCTypes('@@:ffff')
  invocation = NSInvocation.invocationWithMethodSignature(signature)

  invocation.target = CAMediaTimingFunction
  invocation.selector = 'functionWithControlPoints::::'

  [0.4, 0.0, 0.2, 1.0].each_with_index do |point, index|
    pointer = Pointer.new('f')
    pointer[0] = point
    invocation.setArgument(pointer, atIndex: index + 2)
  end

  invocation.invoke

  pointer = Pointer.new('@')
  invocation.getReturnValue(pointer)
  pointer[0]
end
{% endhighlight %}

Eloy suggested a more general purpose workaround [in the mentioned ticket](http://hipbyte.myjetbrains.com/youtrack/issue/RM-123#comment=74-1354).
