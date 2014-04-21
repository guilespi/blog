---
layout: post
title: "Decompiling Clojure II, The Compiler"
date: 2014-04-21 01:03
comments: true
categories: 
- clojure
- compilers
- jvm
---
This is the second post in the Decompiling Clojure series, [in the first post][1] I showed what Clojure looks like in bytecode.

For this entry, I'll do a compiler overview, the idea is to understand why and how does Clojure looks like that.

For other decompilation scenarios you don't usually have the advantage of looking at the compiler internals to guide your decompiling algorithms, so we'll take our chance to peek at the compiler now.

We will visit some compiler source code, so be warned, there's Java ahead.

## It's Java

Well, yes, the Clojure compiler targeting the JVM is written in Java, [there is an ongoing effort][4] to have a Clojure-in-Clojure compiler, but the original compiler is nowhere near of being replaced.

The source code [is hosted on GitHub][2], but the development process is [a little bit more convoluted][3], which means you don't just send pull requests for it, [it was asked for many times][6] and I don't think it's about to change, so if you wanna contribute, [just sign the contributors agreement][5] and follow the rules.

## The CinC Alternative 

The Clojure-in-Clojure alternative is not only different because it's written in Clojure, but because it's built with extensibility and modularization in mind.

In the original Clojure compiler you don't have a chance to extend, modify or use, many of the data produced by the compilation process.

For instance the [Typed Clojure][7] project, which adds [gradual typing][9] to Clojure, [needed a friendlier interface][8] to the compiler analyzer phase. It was first developed by Ambrose Bonnair-Sergeant [as an interface to the Compiler analyzer][10] and then moved to be part of the [CinC analyzer][11].

The CinC alternative is modularized in -at least three- different parts.

* [The analyzer][11], meant to be shared among all Clojure compilers (as Clojurescript)
* [The JVM analyzer][12], contains specific compiler passes for the JVM (for instance locals clearing is done here)
* [The bytecode emitter][13], actually emits JVM bytecode.

There's [a great talk from Timothy Baldridge][19] showing some examples using the CinC analyzer, watch it.

## Compilation process

One of supposed advantages of Lisp-like languages is that the concrete syntax is already the abstract syntax. If you've read some of the [fogus][16] [writings about Clojure compilation][14] tough, he has some opinions on that statement:

{% blockquote fogus http://blog.fogus.me/2012/04/25/the-clojurescript-compilation-pipeline/ %}
This is junk. Actual ASTs are adorned with a boatload of additional information like local binding information, accessible bindings, arity information, and many other useful tidbits.
{% endblockquote %}

And he's right, but there's one more thing, Clojure and Lisp syntax are just serialization formats, mapping to the underlying data structure of the program.

That's why Lisp like languages are easier to parse and unparse, or build tools for them, because the program data structure is accesible to the user and not only to the compiler.

Also that's the reason why macros in Lisp or Clojure are so different than [macros in Scala][18], where the pre-processor handles you an AST that has nothing to do with the Scala language itself.

That's the proper definition of [homoiconicity][17] by the way, the syntax is isomorphic with the AST.

## Compiler phases

In general compilers can be broken up into three pieces

1. Lexer/Parser
2. Analyzer
3. Emitter

Clojure kind of follows this pattern, so if we're compiling a Clojure program the _very_ high level approach to the compilation pipeline would be:

1. Read file
2. Read s-expression
3. Expand macros if present
4. Analyze
5. Generate JVM bytecode

The first three steps are the _Reading_ phase [from the fogus article][15].

There is one important thing about these steps:

Bytecode has **no information about macros whatsoever**, emitted bytecode corresponds to what you see with [macroexpand][20] calls.
Since macros are expanded before analyzing, you shouldn't expect to find anything about your macro in the compiled bytecode, nada, niet, gone.

Meaning, we shouldn't expect to be able to properly decompile macro'ed stuff either.

## Compile vs. Eval 

As said on the first post, the `class` file doesn't need to be on disk, and that's better understood if we think about [eval][23].

When you type a command in the `REPL` it needs to be properly translated to bytecode before the JVM is able to execute it, but it doesn't mean the compiler will save a `class` file, then load it, and only then execute it.

It will be done on the fly.

We will consider three entry points for the compiler, `compile`, `load` and `eval`.

{% img center /images/blog/compiler-reader.png 580 380 'Compiler entry point' 'Compiler entry points' %} 

The `LispReader` is responsible for reading forms from an input stream.

### Compile Entry Point

`compile` is a static function found in the `Compiler.java` file, member of the `Compiler` class, and it does generate a `class` file on disk for each function in the compiled namespace.

For instance it will get called if you do the following in your REPL

```clojure
(compile 'clojure.core.reducers)
```

Clojure function just wraps over the Java function doing the actual work with the signature

{% codeblock compile lang:java https://github.com/clojure/clojure/blob/1.5.x/src/jvm/clojure/lang/Compiler.java#L7162 Compiler.java %}
public static Object compile(Reader rdr, String sourcePath, String sourceName) throws IOException{
{% endcodeblock %}

Besides all the preamble, [the core of the function][21] is just a loop which reads and calls the `compile1` function for each form found in the file.

```java
for(Object r = LispReader.read(pushbackReader, false, EOF, false); r != EOF;
		    r = LispReader.read(pushbackReader, false, EOF, false))
	{
		 compile1(gen, objx, r);
	}
```

As we expect, [the compile1 function does macro expansion][22] before analyzing or emitting anything, if `form` turns out to be a list it recursively calls itself, which is
the `then` branch of the if test:
 
```java
form = macroexpand(form);
if(form instanceof IPersistentCollection && Util.equals(RT.first(form), DO))
  {
	for(ISeq s = RT.next(form); s != null; s = RT.next(s))
	{
		compile1(gen, objx, RT.first(s));
	}
  }
else
  {
	Expr expr = analyze(C.EVAL, form);
        ....
	expr.emit(C.EXPRESSION, objx, gen);
	expr.eval();
  }
```

The `analyze` function we see on the `else` branch does the proper `s-expr` analyzing which emits and evals itself afterwards, more on analyzing ahead.

### Load Entry Point

The `load` function gets called any time we do a `require` for a not pre-compiled namespace.

{% codeblock load lang:java https://github.com/clojure/clojure/blob/1.5.x/src/jvm/clojure/lang/Compiler.java#L7032 %}
public static Object load(Reader rdr, String sourcePath, String sourceName) {
{% endcodeblock %}

For instance, say we do a require for the `clojure.core.reducers` namespace:

```clojure
(require '[clojure.core.reducers :as r])
```

The `clj` file will be read as a stream in the `loadResourceScript` function and passed as the first `rdr` parameter of the `load` function.

You see the `load` function has a pretty similar read form and eval loop as the one we saw in the compile function.

```java
for(Object r = LispReader.read(pushbackReader, false, EOF, false); r != EOF;
r = LispReader.read(pushbackReader, false, EOF, false))
{
  ret = eval(r,false);
}
```

Instead of calling `compile1` calling `eval`, which is our next entry point.

### Eval Entry Point

`eval` is the `e` in REPL, anything to be dynamically evaluated goes through the `eval` function.

For instance if you type `(+ 1 1)` on your REPL that expression will be parsed, analyzed and evaluated starting on the `eval` function.

{% codeblock eval lang:java https://github.com/clojure/clojure/blob/1.5.x/src/jvm/clojure/lang/Compiler.java#L6585 Compiler.java  %}
public static Object eval(Object form, boolean freshLoader)
{% endcodeblock %}

As you see eval receives a `form` by parameter, since knows nothing about files nor namespaces.

`eval` is just straightforward analyzing of the form, and there's not a emit here. This is the _simplified_ version of the function:

```java
form = macroexpand(form);
Expr expr = analyze(C.EVAL, form);
return expr.eval();
```

## The reader

Languages with more complicated syntaxes separate the Lexer and Parser into two different pieces, like most Lisps, Clojure combines these two into just a `Reader`.

The reader is pretty much self contained in `LispReader.java` and its main responsibility is given a stream, return the properly _tokenized_ s-expressions.

The reader dispatches reading to specialized functions and classes when a particular token is found, for instance `(` dispatches to `ListReader` class, digits dispatch to the `readNumber` function and so on.
 
Much of the list and vector reading classes(`VectorReader`, `MapReader`, `ListReader`, etc) rely on the more generic `readDelimitedList` function which receives the particular list separator as parameter.

```java Reader classes for each special character in LispReader
	macros['"'] = new StringReader();
	macros[';'] = new CommentReader();
	macros['\''] = new WrappingReader(QUOTE);
	macros['@'] = new WrappingReader(DEREF);//new DerefReader();
	macros['^'] = new MetaReader();
	macros['`'] = new SyntaxQuoteReader();
	macros['~'] = new UnquoteReader();
	macros['('] = new ListReader();
	macros[')'] = new UnmatchedDelimiterReader();
	macros['['] = new VectorReader();
	macros[']'] = new UnmatchedDelimiterReader();
	macros['{'] = new MapReader();
	macros['}'] = new UnmatchedDelimiterReader();
//	macros['|'] = new ArgVectorReader();
	macros['\\'] = new CharacterReader();
	macros['%'] = new ArgReader();
	macros['#'] = new DispatchReader();
```
This is important because the reader is responsible for reading line and column number information, and establishing a relationship between tokens read and locations in the file.

One of the main drawbacks of the reader used by the compiler is that much of the line and column number information is lost, that's one of the reasons [we saw in our earlier post][1] that for a 7 line function only one line was properly mapped, interestingly, the line corresponding to the outter s-expression.

We will have to modify this reader if we want proper debugging information for our debugger.

## The analyzer

The analyzer is the part of the compiler that translates your `s-expressions` into proper things to be emitted.

We're already familiar with the REPL, in the `eval` function `analyze` and `emit` are combined in a single step, but internally there's a two step process.

First, our parsed but meaningless code needs to be translated into meaningful expressions.

In the case of the Clojure compiler all expressions implement the `Expr` interface:

```java
interface Expr{
	Object eval() ;
	void emit(C context, ObjExpr objx, GeneratorAdapter gen);
	boolean hasJavaClass() ;
	Class getJavaClass() ;
}
```
Much of the [Clojure special forms][24] are handled here, `IfExpr`, `LetExpr`, `LetFnExpr`, `RecurExpr`, `FnExpr`, `DefExpr`, `CaseExpr`, you get the idea.

Those are nested classes inside the Compiler class, and for you visualize how many of those special cases exist inside the compiler, I took this picture for you:

{% img center /images/blog/analyze-expr.png 380 280 'Analyzer' 'Analyzer' %}

As you would expect for a properly modularized piece of software, each expression knows how to parse itself, eval itself, and emit itself.

The analyze function is a switch on the type of the form to be analyzed, just for you to get a taste:

```java
private static Expr analyze(C context, Object form, String name) {

...

if(fclass == Symbol.class)
	return analyzeSymbol((Symbol) form);
else if(fclass == Keyword.class)
	return registerKeyword((Keyword) form);
else if(form instanceof Number)
	return NumberExpr.parse((Number) form);
else if(fclass == String.class)
	return new StringExpr(((String) form).intern());
...
```

And there's special handling for the special forms which are keyed by Symbol on the same file.

```java
IPersistentMap specials = PersistentHashMap.create(
		DEF, new DefExpr.Parser(),
		LOOP, new LetExpr.Parser(),
		RECUR, new RecurExpr.Parser(),
		IF, new IfExpr.Parser(),
		CASE, new CaseExpr.Parser(),
		LET, new LetExpr.Parser(),
		LETFN, new LetFnExpr.Parser(),
		DO, new BodyExpr.Parser(),
		FN, null,
		QUOTE, new ConstantExpr.Parser(),
		THE_VAR, new TheVarExpr.Parser(),
		IMPORT, new ImportExpr.Parser(),
		DOT, new HostExpr.Parser(),
		ASSIGN, new AssignExpr.Parser(),
		DEFTYPE, new NewInstanceExpr.DeftypeParser(),
		REIFY, new NewInstanceExpr.ReifyParser(),
```

Analyze _will_ return a parsed `Expr`, which is now a part of your program represented in the internal data structures of the compiler.

## The bytecode generator 

As said before it uses [ASM][26] so we found the standard code stacking up visitors, annotations, methods, fields, etc.

I won't enter here into specific details about ASM API since it's properly documented somewhere else.

Only notice that no matter if code is eval'ed or not, JVM bytecode _will_ be generated.

## What's next

One of the reasons I ended up here when I started working on the debugger was to see if by any means, I could _add_ better line number references to the
current Clojure compiler.

As said before and as we saw here, the Java Clojure Compiler is not exactly built for extensibility.

The option I had left, was to modify the line numbers and other debugging information at runtime, and that's what I will show you on the next post.

I will properly synchronize Clojure source code with JVM Bytecode, meaning I will synchronize code trees, that way I will not only add proper line references, but I will know
which bytecode corresponds with which `s-expression` in your source.

Doing Clojure I usually end up with lines of code looking like this:

```clojure
(map (comp first rest (partial filter identity)) (split-line line separator))
```

What use do I have for a *line base debugger* with that code??

I want an **s-expression based debugger**, don't you?

One more reason we have to envy [Dr Racket][27], whose debugger already knows about them.

{% img center /images/blog/racket-debug.png 580 480 'Racket Debugger' 'Racket Debugger' %}

Stay tuned to see it working on the JVM.

Meanwhile, [I'm guilespi][25] on Twitter.

[1]: blog.guillermowinkler.com/blog/2014/04/13/decompiling-clojure-i/
[2]: https://github.com/clojure/clojure
[3]: http://dev.clojure.org/display/community/JIRA+workflow
[4]: https://github.com/Bronsa/CinC
[5]: http://clojure.org/contributing
[6]: https://groups.google.com/d/msg/clojure/0gwjKtatf-0/dOMECoHPlM4J
[7]: https://github.com/clojure/core.typed
[8]: http://dev.clojure.org/display/design/Provide+friendly+interface+to+Clojure's+analyzer,+independently+callable+a+la+carte
[9]: http://wphomes.soic.indiana.edu/jsiek/what-is-gradual-typing/
[10]: https://github.com/frenchy64/analyze
[11]: https://github.com/clojure/tools.analyzer
[12]: https://github.com/clojure/tools.analyzer.jvm
[13]: https://github.com/clojure/tools.emitter.jvm
[14]: http://blog.fogus.me/tag/clj-compilation/
[15]: http://blog.fogus.me/2012/04/25/the-clojurescript-compilation-pipeline/
[16]: http://www.twitter.com/fogus
[17]: http://en.wikipedia.org/wiki/Homoiconicity
[18]: http://docs.scala-lang.org/overviews/macros/overview.html
[19]: http://www.youtube.com/watch?v=KhRQmT22SSg
[20]: http://clojuredocs.org/clojure_core/clojure.core/macroexpand
[21]: https://github.com/clojure/clojure/blob/1.5.x/src/jvm/clojure/lang/Compiler.java#L7214-L7221
[22]: https://github.com/clojure/clojure/blob/1.5.x/src/jvm/clojure/lang/Compiler.java#L7138-L7154
[23]: http://clojuredocs.org/clojure_core/1.2.0/clojure.core/eval
[24]: http://clojure.org/special_forms
[25]: http://www.twitter.com/guilespi
[26]: http://asm.ow2.org/
[27]: http://racket-lang.org/
