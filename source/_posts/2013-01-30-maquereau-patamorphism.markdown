---
layout: post
title: "Playing the Fool with Scala Macros - Implementing PataMorphism"
date: 2013-01-30 13:13
comments: true
external-url: 
categories: [scala,scala-macro,functor,morphism,functional programming,pataphysics,uchronic programming]
keywords: scala,scala-macro,functor,morphism,functional programming,pataphysics,uchronic programming
---

<div class="well">
You can find the code on my Github code <a href="http://github.com/mandubian/maquereau">Maquereau</a> 
</div>

# <a name="maquereau">Maquereau _[Scomber Scombrus]_</a>

<div style="float: right"><img src="/images/mandubian/maquereau_white.jpg" /></div>

> _[/makʀo/] in french phonetics_ 

Since I discovered Scala Macros with Scala 2.10, I've been really impressed by their power. But great power means great responsibility as you know. Nevertheless, I don't care about responsability as I'm just experimenting. As if mad scientists couldn't experiment freely!

Besides being a very tasty pelagic fish from scombroid family, [Maquereau](http://github.com/mandubian/maquereau) is my new sandbox project to experiment eccentric ideas with Scala Macros.

Here is my first experiment which aims at studying the concepts of pataphysics applied to Scala Macros.

<br/>
<br/>
# <a name="pataphysics">Pataphysics applied to Macros</a>

### Programming is math

>I've heard people saying that programming is not math.  
>This is really wrong, programming is math.

And let's be serious, how would you seek attention in urbane cocktails without 
 those cute little words such as functors, monoids, contravariance, monads?

> She/He> What do you do?  
> You> I'm coding a list fold.  
> She/He> Ah ok, bye.  
> You> Hey wait…

<br/>

> She/He> What do you do?  
> You> I'm deconstructing my list with a catamorphism based on a F-algebra as underlying functor.  
> She/He> Whahhh this is so exciting! Actually you're really sexy!!!  
> You> Yes I known insignificant creature!

<br/>
### Programming is also a bit of Physics

Code is static meanwhile your program is launched in a runtime environment which is dynamic and you must take these dynamic aspects into account in your code too (memory, synchronization, blocking calls, resource consumption…). For the purpose of the demo, let's accept programming also implies some concepts of physics when dealing with dynamic aspects of a program.

<br/>
### Compiling is Programming Metaphysics

Between code and runtime, there is a weird realm, the compiler runtime which is in charge of converting static code to dynamic program:
  
- The compiler knows things you can't imagine.
- The compiler is aware of the fundamental nature of math & physics of programming.
- The compiler is beyond these rules of math & physics, it's metaphysics.


<br/>
### Macro is Pataphysics

<div style="float: right"><img src="/images/mandubian/ubu.png"/></div>

Now we have Scala Macros which are able:

- to intercept the compiling process for a given piece of code
- to analyze the compiler AST code and do some computation on it
- to generate another AST and inject it back into the compile-chain
 
When you write a Macro in your own code, you write code which runs in the compiler runtime. Moreover a macro can go even further by asking for compilation from within the compiler: `c.universe.reify{ some code }`… Isn't it great to imagine those recursive compilers?

So Scala macro knows the fundamental rules of the compiler. Given compiler is metaphysics, Scala macro lies beyond metaphysics and the science studying this topic is called pataphysics.  

This science provides very interesting concepts and this article is about applying them to the domain of Scala Macros.

I'll let you discover pataphysics by yourself on [wikipedia](http://en.wikipedia.org/wiki/'Pataphysics)

<br/>
> Let's explore the realm of pataphysics applied to Scala macro development by implementing the great concept of patamorphism, well-known among pataphysicians.

<br/>
<br/>
# <a name="defining-patamorphism">Defining Patamorphism</a>

In 1976, the great pataphysician, Ernst Von Schmurtz defined patamorphism as following:

> A patamorphism is a patatoid in the category of endopatafunctors…

Explaining the theory would be too long with lots of weird formulas. Let's just  skip that and go directly to the conclusions. 

#### First of all, we can consider the realm of pataphysics is the context of Scala Macros. 
<br/>
Now, let's take point by point and see if Scala Macro can implement a patamorphism.

#### A patamorphism should be callable from outside the realm of pataphysics
A Scala macro is called from your code which is out of the realm of Scala macro.

#### A patamorphism can't have effect outside the realm of pataphysics after execution
This means we must implement a Scala Macro that :

- has effect only at compile-time
- has NO effect at run-time

From outside the compiler, a patamorphism is an identity morphism that could be translated in Scala as:

{% codeblock lang:scala %}
  def pataMorph[T](t: T): T
{% endcodeblock %}

#### A patamorphism can change the nature of things while being computed

Even if it has no effect once applied, meanwhile it is computed, it can :

- have side-effects on anything
- be blocking
- be mutable

Concerning these points, nothing prevents a Scala Macro from respecting those points.

#### A patamorphism is a patatoid

You may know it but patatoid principles require that the morphism should be customisable by a custom abstract seed. In Scala samples, patatoid are generally described as following:

{% codeblock lang:scala %}
trait Patatoid {
  // Seed is specific to a Patatoid and is used to configure the sprout mechanism  
  type Seed
  // sprout is the classic name
  def sprout[T](t: T)(seed: Seed): T
}
{% endcodeblock %}

So a patamorphism could be implemented as :

{% codeblock lang:scala %}
trait PataMorphism extends Patatoid
{% endcodeblock %}

A custom patamorphism implemented as a Scala Macro would be written as :

{% codeblock lang:scala %}
object MyPataMorphism extends PataMorphism {
  type Seed = MyCustomSeed
  def sprout[T](t: T)(seed: Seed): T = macro sproutImpl
  
  // here is the corresponding macro implementation
  def sproutImpl[T: c1.WeakTypeTag](c1: Context)(t: c1.Expr[T])(seed: c1.Expr[Seed]): c1.Expr[T] = { … }
}
{% endcodeblock %}

But in current Scala Macro API, this is not possible for a Scala Macro to override an abstract function so we can't write it like that and we need to trick a bit. Here is how we can do simply :

{% codeblock lang:scala %}

trait Patatoid{
  // Seed is specific to a Patatoid and is used to configure the sprout mechanism  
  type Seed
  // we put the signature of the macro implementation in the abstract trait
  def sproutMacro[T: c1.WeakTypeTag](c1: Context)(t: c1.Expr[T])(seed: c1.Expr[Seed]): c1.Expr[T]
}

/**
  * PataMorphism 
  */
trait PataMorphism extends Patatoid

// Custom patamorphism
object MyPataMorphism extends PataMorphism {
  type Seed = MyCustomSeed
  // the real sprout function expected for patatoid
  def sprout[T](t: T)(implicit seed: Seed): T = macro sproutMacro[T]

  // the real implementation of the macro and of the patatoid abstract operation
  def sproutMacro[T: c1.WeakTypeTag](c1: Context)(t: c1.Expr[T])(seed: c1.Expr[Seed]): c1.Expr[T] = { 
    …
    // Your implementation here
    … 
  }
}
{% endcodeblock %}

<div class="well">
<h3>Conclusion</h3>
<p>We have shown that we could implement a patamorphism using a Scala Macro.</p>
<p>But the most important is the implementation of the macro which shall:
<ul>
<li>have effect only at compile-time (with potential side-effect, sync, blocking)</li>
<li>have NO effect at runtime</li>
</ul>
<p>Please note that pataphysics is the science of exceptions so all previous rules are true as long as there are no exception to them.</p>
</div>


**Let's implement a 1st sample of patamorphism called `VerySeriousCompiler`.**

<br/>
<br/>
# <a name="veryseriouscompiler">Sample #1: VerySeriousCompiler</a>

## What is it?

`VerySeriousCompiler` is a pure patamorphism which allows to change compiler behavior by :

- Choosing how long you want the compilation to last 
- Displaying great messages at a given speed while compiling
- Returning the exact same code tree given in input

**`VerySeriousCompiler` is an identity morphism returning your exact code without leaving any trace in AST after macro execution.**

`VerySeriousCompiler` is implemented exactly using previous patamorphic pattern and the compiling phase can be configured using custom Seed:

{% codeblock lang:scala %}
/** Seed builder 
  * @param duration the duration of compiling in ms
  * @param speed the speed between each message display in ms
  * @param messages the messages to display
  */
def seed(duration: Long, speed: Long, messages: Seq[String])
{% endcodeblock %}

<br/>
## When to use it?

VerySeriousCompiler is a useful tool when you want to have a coffee or discuss quietly at work with colleagues and fool your boss making him/her believe you're waiting for the end of a very long compiling process.
 
To use it, you just have to modify your code using :

{% codeblock lang:scala %}
VerySeriousCompiler.sprout{
  …some code…
}

//or even 

val xxx = VerySeriousCompiler.sprout{
  …some code returning something…
}
{% endcodeblock %}

Then you launch compilation for the duration you want, displaying meaningful messages in case your boss looks at your screen. Then, you have an excuse if your boss is not happy about your long pause, tell him/her: "Look, it's compiling".

Remember that this PataMorphism doesn't pollute your code at runtime at all, it has only effects at compile-time and doesn't inject any other code in the AST.

<br/>
<br/>
## Usage

### With default seed (5sec compiling with msgs each 400ms)
{% codeblock lang:scala %}
import VerySeriousCompiler._

// Create a class for ex
case class Toto(name: String)

// using default seed
sprout(Toto("toto")) must beEqualTo(Toto("toto"))
{% endcodeblock %}

If you compile:

{% codeblock %}
[info] Compiling 1 Scala source to /workspace_mandubian/maquereau/target/scala-2.11/classes...
[info] Compiling 1 Scala source to /workspace_mandubian/maquereau/target/scala-2.11/test-classes...
Finding ring kernel that rules them all...................
computing fast fourier transform code optimization....................
asking why Obiwan Kenobi...................
resolving implicit typeclass from scope....................
constructing costate comonad....................
Do you like gladiator movies?....................
generating language systemic metafunction....................
verifying isomorphic behavior....................
inflating into applicative functor...................
verifying isomorphic behavior...................
invoking Nyarlathotep to prevent crawling chaos....................
Hear me carefully, your eyelids are very heavy, you're a koalaaaaa....................
resolving implicit typeclass from scope...................
[info] PataMorphismSpec
[info] 
[info] VerySeriousCompiler should
[info] + sprout with default seed
[info] Total for specification PataMorphismSpec
[info] Finished in xx ms
[info] 1 example, 0 failure, 0 error
[info] 
[info] Passed: : Total 1, Failed 0, Errors 0, Passed 1, Skipped 0
[success] Total time: xx s, completed 3 f?vr. 2013 01:25:42
{% endcodeblock %}

<br/>
### With custom seed (5sec compiling with msgs each 400ms)

{% codeblock lang:scala %}
// using custom seed
sprout{
  val a = "this is"
  val b = "some code"
  val c = 123L
  s"msg $a $b $c"
}(
  VerySeriousCompiler.seed( 
    1000L,     // duration of compiling in ms
    200L,       // speed between each message display in ms
    Seq(        // the message to display randomly 
      "very interesting message", 
      "cool message"
    )
  )
) must beEqualTo( "msg this is some code 123" )
{% endcodeblock %}

If you compile:

{% codeblock %}
[info] Compiling 1 Scala source to /workspace_mandubian/maquereau/target/scala-2.11/classes...
[info] Compiling 1 Scala source to /workspace_mandubian/maquereau/target/scala-2.11/test-classes...
toto..........
coucou..........
toto..........
coucou..........
[info] PataMorphismSpec
[info] 
[info] VerySeriousCompiler should
[info] + sprout with custom seed
[info] Total for specification PataMorphismSpec
[info] Finished in xx ms
[info] 1 example, 0 failure, 0 error
[info] 
[info] Passed: : Total 1, Failed 0, Errors 0, Passed 1, Skipped 0
[success] Total time: xx s, completed 3 f?vr. 2013 01:25:42
{% endcodeblock %}

<br/>
<br/>
## Macro implementation details

The code can be found on Github [Maquereau](http://github.com/mandubian/maquereau).

Here are the interesting points of the macro implementation.

#### Modular macro building

We use the method described on [scalamacros.org/writing bigger macros](http://docs.scala-lang.org/overviews/macros/overview.html#writing_bigger_macros)

{% codeblock lang:scala %}
abstract class VerySeriousCompilerHelper {
  val c: Context
  …
}

def sproutMacro[T: c1.WeakTypeTag](c1: Context)(t: c1.Expr[T])(seed: c1.Expr[Seed]): c1.Expr[T] = {
  val helper = new { val c: c1.type = c1 } with VerySeriousCompilerHelper
  helper.sprout(t, seed)
}
{% endcodeblock %}

<br/>
#### input code evaluation in macro

The Seed passed to the macro doesn't belong to the realm of Scala Macro but to your code. In the macro, we don't get the Seed type but the expression Expr[Seed]. So in order to use the seed value in the macro, we must evaluate the expression passed to the macro:

{% codeblock lang:scala %}
val speed = c.eval(c.Expr[Long](c.resetAllAttrs(speedTree.duplicate)))
{% endcodeblock %}

*Please note that this code is a work-around because in Scala 2.10, you can't evaluate any code as you want due to some compiler limitations when evaluating 
an already typechecked tree in a macro. This is explained in this [Scala issue](https://issues.scala-lang.org/browse/SI-5464)*

<br/>
#### input code re-compiling before returning from macro

We don't return directly the input tree in the macro even if it would be valid with respect to patamorphism contract.
But to test Macro a bit further, I decided to "*re-compile*" the input code from within the macro. You can do that using following code:

{% codeblock lang:scala %}
reify(a.splice)
{% endcodeblock %}

<br/>
<br/>
# <a name="conclusion">Conclusion</a>

This article shows that applying the concepts of pataphysics to Scala Macro is possible and can help creating really useful tools. 

The sample is still a bit basic but it shows that a patamorphism can be implemented relatively simply. 

I'll create other samples and hope you'll like them!

Have patafun!