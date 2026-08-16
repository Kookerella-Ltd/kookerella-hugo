---
title: "Functional XSLT/XPath/XQuery - #2 Functor pattern"
date: 2024-06-03
slug: "functional-xsltxpathxquery-2-functor"
tags: ["xslt", "xpath", "functional-programming", "functor", "xquery"]
---

# Functional XSLT/XPath/XQuery - #2 Functor pattern

In the previous blog we introduced some machinery, the id function, and function composition, we talked about function types, and invented an informal notion of parametric types in order to aid our thinking.

We now move onto the functional patterns. These are a bit like OO patterns but have more formal mathematical underpinnings. Knowing what to call the things is problematic in itself, in mathematics these underpinnings are called a *category,* in functional circles they are often called a *typeclass ,* this is a borrowed term from Haskell that has a particular mechanism for implementing them which may be misleading, other languages have different mechanisms, some languages have no formal mechanism, but have informal mechanisms, or no mechanism at all apart from idiomatic usage, like an OO pattern.

We wont get too hung up on this, we will try to use pattern or idiom to mean an informal usage, we will try to use category to mean the underlying formal theory, and we will introduce a suitable mechanism based on Map to allow slightly more structured usage later on.

### #2 - The Functor pattern

**#2.1 Sequence**

The good news here is you already know this pattern, you already use it, implicitly and explicitly. Its most familiar in the context of mapping elements of a sequence.

Consider the following XSLT code (using any xml input):

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <xsl:template match="/">
        <output>
            <xsl:for-each select="('hello','world')">
                <xsl:sequence select="string-length(.)"/>                
            </xsl:for-each>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

unsurprisingly this will output:

```xml
<output>5 5</output>
```

We can equivalently write this code:

```xml
    <xsl:template match="/">
        <output>
            <xsl:sequence select="for $x in ('hello','world') return string-length($x)"/>
        </output>
    </xsl:template>
```

here we're using a XQuery style `for` expression to loop through a sequence of string and convert them, or equivalently using the `!` operator:

```xml
    <xsl:template match="/">
        <output>
            <xsl:sequence select="('hello','world') ! string-length(.)"/>
        </output>
    </xsl:template>
```

All these things do the same thing, we could if we wanted write our own version of this code like this:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:sequence="http://kookerella.com/xsl:sequence">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="sequence:map" as="item()*">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="sequence" as="item()*"/>
        <xsl:sequence select="$sequence ! $mapper(.)"/>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:sequence select="sequence:map(string-length#1,('hello','world'))"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

Some of this may be unfamiliar, we saw with functional composition that we could pass functions as parameters, and we do this here, passing in `string-length#1` (which is the function `string-length` that takes a single parameter).

This use of passing functions into functions is called *higher order functions,* its a foundational technique of functional programming, its already part of and used by the XPath libraries, but its important to understand the technique, initially to be able to write client code that calls functions that require functions as parameters, and then, when appropriate, writing functions that take parameters that are functions.

Note also, if you look at the type of the function `string-length` you find its this (well almost):

`fn:string-length`(`$arg as xs:string`) `as xs:integer`

If the input to the `sequence:map` is sequence of `xs:string`, then the output is going to be a seqence of `xs:integer`.

As we did with the `id` function previously we can mentally think of a psuedo type for our map function, lets first write it in XPath:

```xml
    <xsl:variable name="sequence:map" 
        as="function(function(item()*) as item()*,item()*) as item()*">
        <xsl:sequence select="function($mapper,$sequence) {$sequence ! $mapper(.)} "/>
    </xsl:variable>

    <xsl:template match="/">
        <output>
            <xsl:sequence select="$sequence:map(string-length#1,('hello','world'))"/>
        </output>
    </xsl:template>
```

We can now see that function type is

```xml
function(function(item()*) as item()*,item()*) as item()*
```

but its psuedo parametric type is:

```xml
function(function(A) as B,A*) as B*
```

(I've taken some liberties with the dropping \*s where its convenient for a simple explanation of the theory).

So we have a function that takes a function that maps things of type A to type B, and given a sequence of things of type A, we can construct a sequence of things of type B.

That is the essence of the Functor pattern, we take a function from A to B, some data structure defined in terms of A, and we produce a data structure defined in terms of B.

This is pretty inherent to much of XPath etc, XML is a hierarchy of sequences, and we map the values inside those sequences to new values in sequences, but we can and will do it with maps, arrays, trees, and other data structures which we haven't defined yet.

Let's make this a bit more formal, (if this is too tricky, you can skip it, especially on a first reading, just be happy that sequence is a functor).

The formal definition for Functor is that the data type (here sequence) must have a function map, but it must obey the rules, and the rules expressed as equivalences (here `==` is equivalience):

* `sequence:map(core:id#1,$sequence) == $sequence`

i.e. if we map the contents of a sequence using our foundational id function, is the same as doing nothing, i.e. doing nothing to every element in the sequence in turn is the same as doing nothing to the sequence.

That seems believable!

and here is the XPath function parameter placeholder):

* `sequence:map(function:compose($f,$g),?) == function:compose(sequence:map($f,?),sequence:map($g,?))`

ouch (unfortunately XPath isn't particularly succinct for these sorts of things).

So, this says, if we map a sequence over the composition of two functions f and g, that's the same as composing the map of a sequence over the function f with the map of a sequence over the function g, or maybe more intuitively....

if we want to do 2 things in turn to each element in a sequence we can either loop through the sequence one by one, doing the 2 things in turn OR we can loop through the sequence doing the first thing, then loop through the resulting sequence doing the second.

Again, like the previous law, this is also true of our sequence type, so, a category theorist would say the function `sequence:map` and the sequence data type together are an instance of the category called functor, a Haskell programmer would says its an instance of the typeclass Functor. Note its the data type **and** the function that form the Functor, for some of these idioms there may be more than one way to construct the category/typeclass, here actually there isn't, and in fact for any data type there is only one way to construct the Functor.

Note, when looking at XPath library function you'll notice they tend to have 'forgiving' signatures e.g.

`fn:string`(`$arg as item()?`) `as xs:string`

it forgives the programmer if you pass in an empty parameter, it would be simpler not to allow this, and insist on the stricter `$arg as item()` but in their wisdom the language designers have decided not to do that (I suspect they had good reasons for it at the time).

This is out of step with the usual FP (functional programming) practice which would have the stricter signature but give you tools to handle this situation, the missing tool here is the `Maybe` data type which we will introduce later. We will in general try to ignore these weaker signatures and pretend they don't exist, in fact this has very little practical impact, personally I wouldn't want to ever pass an empty sequence as an argument, and if I did it would be a mistake and quite possibly a bug.

The key message is, don't let this confuse you when trying to identify the parametric type of a function, both `?` and `*` can confuse the issue, and we have to try to work around their idiosyncrasies.

XPath also *appears* to support function types like this:

`fn:string`() `as xs:string`

with this specification, this looks like a function that takes no parameters and returns a string, but it isn't, the specification says:

`In the zero-argument version of the function, $arg defaults to the context item. That is, calling fn:string() is equivalent to calling fn:string(.).`

So the signature is at best misleading and at worst a dangerous lie, again I suspect the languages designers had good reasons to allow this, the cost though is the language is (to me) at times irregular and confusing, so beware.

The key message is don't let these idiosyncrasies deflect you.

Sequence isn't the only Functor we can discover, lets try array.

**#2.2 Array as a Functor**

Array is very similar to sequence, and arguably more regular, so lets dive straight in:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:array="http://www.w3.org/2005/xpath-functions/array"
                xmlns:xarray="http://kookerella.com/xarray">
    <xsl:output method="xml" indent="yes"/>

    <!-- array:map as function(function(A) as B,array(A)) as array(B) -->
    <!-- array:map :: ((A -> B),array A) -> array B -->
    <xsl:function name="xarray:map" as="array(*)">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="array" as="array(item()*)"/>
        <xsl:sequence select="array:for-each($array,function($a) { $mapper($a) })"/>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:variable name="arrayOfLengths" as="array(xs:integer)" select="xarray:map(string-length#1,array { ('hello','world') })"/>
            <xsl:sequence select="$arrayOfLengths"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

note here, we've can't use the `array` namespace to define our version of map, so we've created a new one `xarray` and we find that `array:for-each` pretty much does what we require.

Also note we will put pseudo types before our functions to help us understand what's actually happening.

So here the data type array and the function `xarray:map`, form an instance of Functor.

Hopefully now we can see the attraction of these idioms, both already exist in XPath, why would we want to recreate them, but its clear now that where XPath/XSLT has `for` , `!` , `/` and `array:for-each` and `xslt:for-each` we have just have `map` (in the two namespaces) this is convenient, why learn 3 things when you can learn 1? Later on we will see how we can utilise this uniformity of structure more explicitly.

**#2.3 Map as Functors**

Map is actually a more complex beast than array and sequences, not because it does more 'complex' stuff but because its pseudo type is more complex.

We can declare an array of integers, or a sequence of strings, but when we declare a map we can have a map of string keys, with integer values, or a map of integer keys with array values. So when we glibly define a functor as "we take a function from A to B, some data structure defined in terms of A, and we produce a data structure defined in terms of B", then what is A in a map? is it the type of the key or the type of the value. In fact we can construct a functor using either of these decisions.

Its more usual to construct the default functor of the map over the value type, that just tends to me more useful, but both constructions are valid, and both deserve to exist, if they both exist then there is a question of naming, and here there are two aspects to consider

* is it enough that these idioms/patterns are just that, mental models of usage, like OO patterns, where we try to use a common language to aid both our communication about our code and give us the mental building blocks to construct our code in a uniform way, even if sometimes the names are slightly different....or.....
* is this something more? are these idioms more concrete, more fixed, are we going to utilise explicitly common structure and common naming?

Well the answer is both, we WILL utilise these idioms in a quite explicit manner later on, but to introduce the mechanism by which we are going to do this, is premature. The machinery will obscure what is probably not a completely simple straightforward narrative. This is a genuine issue in the design of functional languages and different languages use different approaches, but they all (in my experience) start at the position that you define standalone uniquely defined functions, which, if you are sensible, you will try to label in an idiomatic manner, and then later allow you to construct more explicitly uniform constructs to satisfy the second requirement.

So, we will construct two functors here, one based on the key, and one based on the value, here we go:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:map="http://www.w3.org/2005/xpath-functions/map"
                xmlns:xmap="http://kookerella.com/xmap">
    <xsl:output method="xml" indent="yes"/>

    <!-- xmap:mapValue as function(function(A) as B,map(Key,A)) as map(Key,B) -->
    <!-- xmap:mapValue :: ((A -> B),map Key A) -> map Key B -->
    <xsl:function name="xmap:mapValue" as="map(*)">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="map:merge(map:for-each($map,function($key,$value) { map { $key : $mapper($value) } }))"/>
    </xsl:function>

    <xsl:function name="xmap:toArray" as="array(*)">
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="array { map:for-each($map,function($key,$value) { array { $key,$value } }) }"/>
    </xsl:function>
        
    <xsl:template match="/">
        <output>
            <xsl:variable 
                name="helloWorldLengthsAsIntegers" 
                as="map(xs:string,xs:integer)" 
                select="
                    map {
                        'hello' : 5,
                        'world' : 5
                    }"/>
            <xsl:variable name="helloWorldLengthsAsStrings" as="map(xs:string,xs:string)" select="
                xmap:mapValue(xs:string#1,$helloWorldLengthsAsIntegers)"/>
            <xsl:sequence select="xmap:toArray($helloWorldLengthsAsStrings)"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

Here we define the function `mapValue`, it maps the data type `map(Key,A)` to the type `map(Key,B)`, i.e. the data type is a map with a fixed key defined over A is mapped to a map with a fixed key defined over B, this defines a Functor instance.

(we've had to include a utility function `xmap:toArray` to serialise the output into the XML)

Similarly we can define `xmap:mapKey` in a very similar way to define another Functor instance for the map with a fixed value type.

This case where a type is parameterised in two types is not uncommon and so there is a special category called Bifunctor which captures this notion, it could be defined using 2 map functions, one on either type, but its more commonly defined with a single function bimap like this:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:map="http://www.w3.org/2005/xpath-functions/map"
                xmlns:xmap="http://kookerella.com/xmap"
                xmlns:function="http://kookerella.com/function">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="function:id">
        <xsl:param name="x"/>
        <xsl:sequence select="$x"/>
    </xsl:function>

    <!-- xmap:bimap as function(function(KeyA) as KeyB,function(ValueA) as ValueB,map(KeyA,ValueA)) as map(KeyB,ValueB) -->
    <!-- xmap:bimap :: ((KeyA -> KeyB),(ValueA -> ValueB),map KeyA ValueA) -> map KeyB ValueB -->
    <xsl:function name="xmap:bimap" as="map(*)">
        <xsl:param name="mapKey" as="function(item()*) as item()*"/>
        <xsl:param name="mapValue" as="function(item()*) as item()*"/>
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="map:merge(map:for-each($map,function($key,$value) { map { $mapKey($key) : $mapValue($value) } }))"/>
    </xsl:function>

    <!-- xmap:mapValue as function(function(A) as B,map(Key,A)) as map(Key,B) -->
    <!-- xmap:mapValue :: ((A -> B),map Key A) -> map Key B -->
    <xsl:function name="xmap:mapValue" as="map(*)">
        <xsl:param name="mapValue" as="function(item()*) as item()*"/>
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="xmap:bimap(function:id#1,$mapValue,$map)"/>
    </xsl:function>

    <!-- xmap:mapKey as function(function(A) as B,map(A,Value)) as map(B,Value) -->
    <!-- xmap:mapKey :: ((A -> B),map A Value) -> map B Value -->
    <xsl:function name="xmap:mapKey" as="map(*)">
        <xsl:param name="mapKey" as="function(item()*) as item()*"/>
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="xmap:bimap($mapKey,function:id#1,$map)"/>
    </xsl:function>

    <xsl:function name="xmap:toArray" as="array(*)">
        <xsl:param name="map" as="map(*)"/>
        <xsl:sequence select="array { map:for-each($map,function($key,$value) { array { $key,$value } }) }"/>
    </xsl:function>
        
    <xsl:template match="/">
        <output>
            <xsl:variable 
                name="helloWorldLengthsAsIntegers" 
                as="map(xs:string,xs:integer)" 
                select="
                    map {
                        'hello' : 5,
                        'world' : 5
                    }"/>
            <xsl:variable name="helloWorldLengthsAsStrings" as="map(xs:string,xs:string)" select="
                xmap:mapValue(xs:string#1,$helloWorldLengthsAsIntegers)"/>
            <xsl:sequence select="xmap:toArray($helloWorldLengthsAsStrings)"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

Note we have now implemented mapKey and mapValue in terms of bimap.

Next time we will introduce a new data type, and discuss some of the wrinkles in these technologies that make using Sequence as your default data type all the time problematic.

**#2.4 Functions as Functors**

Yes, surprising isn't it.

Functions can be made into Functors based on their return types, the situation is similar to the Map scenario where we have two parameters, i.e. we think about functions in general as having the type:

function(A) as B

If we fix the input type, then we can map arbitrary functions based on their return type. Lets just for the sake of simplicity call the type of all functions that take an integer as an input as functionInteger, so a functionInteger that returns a string is of type

functionInteger(string)

Ok, thats a bit weird, (its just a thought experiment), but lets write down the types of our other map functions, for array we have.

`function(function(A) as B,xs:array(A)) as xs:array(B)`

`function(function(A) as B,sequence(A)) as sequence(B)`

so lets try

`function(function(A) as B,functionInteger(A)) as functionInteger(B)`

hmmm, that seems to work its just

`function(function(A) as B,function(xs:integer) as A) as function(xs:integer) as B`

but there's nothing special about xs:integer here, this works for any type so

`function(function(A) as B,function(C) as A) as function(C) as B`

WAIT!!!!

we've seen this signature before...its functional composition.

we get it for free.

```xml
    <!-- (B -> C) -> (A -> B) -> (A -> C) -->
    <!-- function(function(B) as C,function(A) as B) as function(A) as C -->
    <xsl:function name="function:map" as="function(item()*) as item()*">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="func" as="function(item()*) as item()*"/>
        <xsl:sequence select="function:compose($mapper,$func)"/>
    </xsl:function>
```
