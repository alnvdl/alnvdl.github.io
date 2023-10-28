---
layout: post
title: "Generating text and code with fstringen"
date: "2023-09-07 14:00:00 -0300"
excerpt: Or the tale of how I bent Python in dangerous and obscure ways.
---

Automating repetitive text or code generation is a problem that many programmers eventually face. Generally, there are two parts to this problem: **modeling** your ideas, and then **generating** code or text from them.

On the modeling side, DSLs and specifications like OpenAPI focus on standardizing on a common structure for expressing ideas in a given domain (e.g, web APIs in the case of OpenAPI). On the generation side, the most common idea is using templating languages. Stretching the argument a bit, generics, as present in several programming languages, can ultimately be seen as a way of automating code generation based on a model defined by the language itself.

One could say compilers are solving just this problem: they take a model (code in the programming language you are using) and generate some other artifact (code in another programming language, machine code, etc). Compilers will typically parse their input, build an internal representation of the language (e.g., an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)), and then use that to generate the target code.

But not everyone has the need, time or skills to formally define a language and write a compiler, even if using tools like [Lex](https://en.wikipedia.org/wiki/Lex_(software)) and [Yacc](https://en.wikipedia.org/wiki/Yacc). More often than not, all we can afford is quick-and-dirty :)

In those cases, we usually take an input **model** that's already structured (e.g., a JSON model or a dictionary) and then **browse** that model to **generate text** or code. That's the lazy approach this post will cover.

To go with a lazy approach like this, we have 3 problems to solve:

1. **Defining the model**
2. **Browsing the model in generators**
3. **Writing generator code**

---

Python introduced f-strings with Python 3.6, back in 2016. When I first saw f-strings, they reminded me of a templating language. More interestingly, triple-quoted multi-line strings could be made into f-strings. That's the beginning of a promising templating language!

That got me thinking I could perhaps develop something that would help in solving the three problems listed above.

1. **Defining the model**: Python has dictionaries, and great libraries that create dictionaries based on JSON or YAML input.
2. **Browsing the model in generators**: that needs to be developed.
3. **Writing generator code**: Multi-line triple-quoted f-strings!

**Problem #1 is solved from the start.**

For problem #2: I always wished we could more easily browse Python dictionaries using paths as if they were a filesystem, and not individual discrete objects.

We could borrow the long-established, somewhat obvious path syntax: `/firstKey/anotherKey/2/subKey`. That can easily be understood as `mydict["firstKey"]["anotherKey"][2]["subKey"]`.

Browsing a dictionary or enumerable object (e.g., a list or tuple) in Python, you can have one of three outcomes:
- You get a dictionary, and you can continue the lookup by key for another level;
- You get an enumerable object, and you can continue the lookup by index for another level;
- You get a value that is not a dictionary or enumerable and you can't browse any deeper.

So let's call that dictionary or enumerable a `Model`. And for browsing models using a path, let's call that operation `select`.

To make `Model`s easier to use, it would be nice if whenever we are selecting a model, we always get a new `Model` as the result. Something like this:
```py
mymodel = Model("mymodel", {
    "goto": "/firstKey/anotherKey/2/subKey",
    "firstKey": {
        "anotherKey": [None, None, {"subKey": "it works!"}]
    }
})
m1 = mymodel.select("/firstKey") # Holds value: {"anotherKey": [...]}
hasattr(m1, "select") # True
m2 = m1.select("anotherKey/0") # Holds value: None
hasattr(m3, "select") # True
m3 = mymodel.select("goto->") # Holds value: "/firstKey/anotherKey/2/subKey"
hasattr(m3, "select") # True
```

When you instantiate a `Model` and use `select`, you always get a new `Model`. That's where Python metaclasses proved to be very useful: the `select` method in the `Model` object creates a new class for every sub-model, inheriting from whatever type that value had, but adding the `select` method ([and a few other ones](https://github.com/alnvdl/fstringen#using)).

Finally, for some models, such as OpenAPI, we need links or [references](https://spec.openapis.org/oas/latest.html#reference-object) between parts of the model. Taking the `mymodel` example above, the `goto` key points to a string with value `/firstKey/anotherKey/2/subKey`. It should be possible to select `goto->` and get the value at `/firstKey/anotherKey/2/subKey`.

**Now we have a nice way of browsing models in generator functions, so problem #2 is solved!**

For problem #3, things are a bit more complicated. I knew I wanted to use fstrings, but Python triple-quoted strings are a bit finicky. For example, if you have the following:
```py
def mystring():
    return """
    a line
    another line
    """
```

Calling `mystring()` returns `\n    a line\n    another line\n    `. Notice the includes the indentation and newlines.

Instead, I wanted the behavior we imagine the first time we accidentally use triple-quoted strings: `mystring()` should return `a line\nanother line`, as if the function indent didn't count.

To do that, I invented something I called fstringstars: a triple-double-quoted string that starts and ends with a `*`. For example, the following is a fstringstar:
```py
myfstringstar = """*
some text
*"""
```

In order for those strings to behave differently from Python's regular f-strings, I had to hack Python in a very questionable way: I introduced the `gen` [decorator](https://github.com/alnvdl/fstringen/blob/d7df5d624e6268675b43cd55cc3e1db661cd16ca/fstringen/generator.py#L151-L212) that when used on a function, takes the function code, replaces all fstringstars by a [function call](https://github.com/alnvdl/fstringen/blob/d7df5d624e6268675b43cd55cc3e1db661cd16ca/fstringen/generator.py#L117-L132).

That function receives the fstringstar and returns a string that behaves how I wanted triple-quoted strings to work.

Additionally, around that time, I was also learning about React's [JSX](https://react.dev/learn/writing-markup-with-jsx). They have a nice feature where if you pass a list of elements, you get those elements inserted into the parent element at the right spot. In other words, I wanted this code:
```py
def mystring():
    a = [1, 2, 3]
    return """
    a line
    {a}
    """
```

To return `a line\n1\n\2\n3`.

To do that, I went further into the rabbit hole of [function code manipulation](https://github.com/alnvdl/fstringen/blob/d7df5d624e6268675b43cd55cc3e1db661cd16ca/fstringen/generator.py#L16-L51) using generators.

**That's it, I believe problem #3 is now solved!**

---

So through some serious Python hackery, I was able to implement a model-based generator using triple-quoted f-strings!

Recapping, the hacks were:

- Rewriting decorated functions on-the-fly using the [`inspect` module](https://docs.python.org/3/library/inspect.html);
- Dedenting triple-quoted f-strings in those decorated functions using regexes;
- Introducing a suspiciously magic way of browsing Python dictionaries and lists that involves an even more awkward usage than usual of Python's [metaclasses](https://docs.python.org/3/reference/datamodel.html#metaclasses) feature;

This is how [fstringen](https://github.com/alnvdl/fstringen) was born. It has no dependencies and (I hope) good enough informal documentation in its [README](https://github.com/alnvdl/fstringen#readme). Considering all the weird things it does, I wouldn't personally use it in anything other than code that's purely acting as a generator for code or text.

fstringen was sparked by learning about f-strings in Python. Learning React and JSX also made me think about how I could imitate the way React produces a component tree from JavaScript functions or classes, but with Python functions producing text or code instead. The path syntax and the `select` operation is inspired by both OpenAPI and also by a project I participated in a previous job a few years ago, where some really smart engineers used the TCL language to model network devices and generate C code. If you know TCL, you are probably laughing at this post and its complexity.

Since I wrote fstringen, I've used to implement a couple of (non-open-source) model-generators: an extremely barebones subset of [Confluent's Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html) and a tool for automating the generation of certain reports about projects. It should be quite easy to use it for generating API code based on OpenAPI specs as well, but I haven't tried it yet.
