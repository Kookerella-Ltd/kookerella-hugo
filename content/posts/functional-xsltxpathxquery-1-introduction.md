---
title: "Functional XSLT/XPath/XQuery - #1 Introduction"
date: 2024-06-03
slug: "functional-xsltxpathxquery-1-introduction"
tags: ["xslt", "xpath", "xquery", "functional-programming"]
---

# Functional XSLT/XPath/XQuery - #1 Introduction

To the modern programmer these languages appear strange beasts, born when the world was obsessed with XML, it is declarative, almost functional, but somehow missing some of the idioms of later more consciously functional languages. For a functional programmer it misses familiar function and data types that the functional programmer instinctively expect. To be fair their roots predate much of the flowering of functional patterns in language development and whilst the semantics of the languages are quite well suited to this style of development, the foundations are idiosyncratic, and now locked in by their history to be slightly out of step.

Maybe you're an XSLT developer who wants to learn and utilise some functional programming techniques, or maybe, like me, you're a functional programmer, who dabbles in these technologies but finds that constructing clean simple software is just beyond your reach.

This blog does expect you to know a bit of XPath/XSLT/XQuery, but there are better introductions to these languages elsewhere, but it doesn't expect you to be an expert, I'm far from expert myself (as will become apparent in this blog). This blog doesn't expect you to know anything though about functional programming, and whilst it isn't intended to be functional programming course, it will try to ease you into some functional patterns through examples, rather than bamboozle you with theory, or maths.

(Be aware that this is a work in progress, I'm not simply transferring some prewritten script into a blog, or publishing some complete library of code, the code I write here will be added to a public github repository for you to see and contribute, its first iteration will be focussed on XSLT usage, but later I hope to expend it to XQuery usage).

### Series Contents

* Part 1 - Foundational tools
    * Introduction (above)
    * The id function
    * Function composition
* Part 2 - Functor pattern
    * Maybe data type
* Part 3 - Applicative
* Part 4 - Monad
* Part 5 - Traversable
* Part 6 - A deferred list

### #1 - Foundational tools

**#1.1 - the id function**

Functional programming is about functions, and, well...data, but most of the focus is on the functions. The first missing tool from the X language toolbox is the 'id' function (not to be confused with the existing `fn:id`, which is a very different thing).

We will, for all examples, try to give simple complete examples that you should be able to run, they will initially utilise the below input xml file (rather than use initial templates which are more idiosyncratic), and a hopefully simple XSLT file.

We will drop the xml header, here is our first xml input file:

```xml
<input/>
```

With XSLT files, we will drop the header, and try to show the simplest possible code we can:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:core="http://kookerella.com/function">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="function:id" as="item()*">
        <xsl:param name="value" as="item()*"/>
        <xsl:sequence select="$value"/>
    </xsl:function>
        
    <xsl:template match="/">
        <output>
            <xsl:sequence select="function:id(('hello','world'))"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

So here we a hello world program that takes the string `'hello world'`, passes it to a function, that returns it unchanged, and its then output.

This is clearly a silly example, we would never actually want to do this, but the `id` function is useful in the theory of what we do, even if its direct application is rare.

There are stylistic things here to note:

* we've declared a namespace `core` in which we will put foundational constructs, we will in this blog use namespaces to help modularise our labelling.
* we will use explicit type (`as`) declarations to constrain our declarations where we can to make their meaning as explicit as we can to both the runtime but also to us.
* we use `item()*` as our root type, this may seem odd, it may feel more natural to use `item()`, but actually `item()*` is a more lenient type, the code above doesn't run with `item()` , XSLT's base data type is the sequence not the object or value.

Alternatively could have written this

```xml
    <xsl:variable 
        name="function:id" 
        as="function(item()*) as item()*" 
        select="function($value) { $value }"/>

    <xsl:template match="/">
        <output>
            <xsl:sequence select="$function:id(('hello','world'))"/>
        </output>
    </xsl:template>
```

The behaviour of this code is identical, but the implementation is subtly different:

* we're defining a variable, a value that has a type `function(item()*) as item()*`
* the function is defined in XPath not in XSLT
* we're invoking the function via the variable rather than directly using the name of the function.

We will define functions using both styles, as appropriate, but the XPath style has the advantage of allowing us to talk explicitly about the type of the function variable, `function(item()*) as item()*` and function types will become more and more important to understand as we move through the tutorial.

Notice though that this function type is quite weak, it says we pass any sequence, and we get any sequence back, but we could in theory be more precise, if we pass some data of type `xs:string*`, we will get data of type `xs:string*`, if we pass an `array(xs:integer?)?` in, we will get an `array(xs:integer?)?` back, XPath isn't powerful enough to capture this notion, we need parametric types, but its useful to think of the type of id as a **psuedo parametric type** e.g.

`function(TypeA) as TypeA`

this isn't a valid XPath type, this is simply a construct we've invented for this blog, don't put it into your code, but thinking about how the types are transformed is important in functional patterns, we can equivalently think of this type as:

`function(A) as A` or `function(Ham) as Ham` or `function(Foo) as Foo`

the type parameter label itself, like variable names in code, is irrelevant. In this case, the type of the function uniquely defines the function, the only function that has this type is the id function, all other functions that correctly implement it, are equivalent.

**#1.2 - function composition**

Our second foundational tool is known as function composition. In functional languages, functions are the atoms, and functional composition, is the foundational mechanism to glue functions together to make new functions.

Consider this template:

```xml
    <xsl:template match="/">
        <output>
            <xsl:variable name="birthday" as="xs:date" select="xs:date('2020-01-01')"/>
            <xsl:sequence select="xs:string(day-from-date($birthday))"/>
        </output>
    </xsl:template>
```

here we simply take a date, extract the day from it, and convert it to a string.

Consider this implementation though:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:function="http://kookerella.com/xsl:function">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="function:then" as="function(item()*) as item()*">
        <xsl:param name="f" as="function(item()*) as item()*"/>
        <xsl:param name="g" as="function(item()*) as item()*"/>
        <xsl:sequence select="function($value) { $g($f($value)) }"/>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:variable name="birthday" as="xs:date" select="xs:date('2020-01-01')"/>
            <xsl:sequence select="function:then(day-from-date#1,xs:string#1)($birthday)"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

This is the equivalent code, again, as with our id example, we wouldn't implement this example like this, but this is simply to illustrate what we can do, we have introduced a new function that takes two parameters, both of which are functions, and returns a new function that takes a parameter and applies the first function to this parameter and then applies the second function to the result, i.e. it applies the first parameter first **then** the second.

For the moment it isn't especially important to understand this in depth, or feel confident to use it, the point is to understand that this **can** be done and we're simply applying the first then the second function.

Irritatingly (to me) the mathematicians who discovered this notion, decided to follow the syntax of the expression rather than the syntax of the natural language description and they define compose the other way around.

```xml
    <xsl:function name="function:compose" as="function(item()*) as item()*">
        <xsl:param name="f" as="function(item()*) as item()*"/>
        <xsl:param name="g" as="function(item()*) as item()*"/>
        <xsl:sequence select="function($value) { $f($g($value)) }"/>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:variable name="birthday" as="xs:date" select="xs:date('2020-01-01')"/>
            <xsl:sequence select="function:compose(xs:string#1,day-from-date#1)($birthday)"/>
        </output>
    </xsl:template>
```

Some people instinctively prefer this definition, rather than the previous one, (I don't), but I will use the traditional `compose` in preference because function composition is the generally accepted phrase across different languages and academic disciplines and will make reading around the subject easier.

Now think about the psuedo parametric type of this function, its much more complex than the id function. First, lets write out an implementation in pure XPath:

```xml
    <xsl:variable 
        name="function:compose" 
        select="function($f,$g) { function($value) { $f($g($value)) } }" 
        as="function(function(item()*) as item()*,function(item()*) as item()*) as function(item()*) as item()*"/>
```

The type here seems horrific, but if we think about the types, we can work it out.

If the type of parameter the parameter is A say, then the first function that is applied to it, must take an A, and lets say it returns a B, so the second function that is applied to it must take a B and return a C, so the type is:

`function(function(B) as C,function(A) as B) as function(A) as C`

(other programming languages that support this sort of thing, tend to have much more succint type languages and the above function would be written in some like this: `((B -> C),(A -> B)) -> (A -> C)` which reads, give a function from B to C and a function A to B we can return a function A to C)

In the next blog we will visit the first functional idiom, which I suspect will seem very familiar to you, even if you didn't realise it.
