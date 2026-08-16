---
title: "Functional XSLT/XPath/XQuery - #4 Monoid"
date: 2024-06-09
slug: "functional-xsltxpathxquery-4-monoid"
tags: ["xslt", "xpath", "xquery", "functional-programming", "monoids"]
---

# Functional XSLT/XPath/XQuery - #4 Monoid

Usually in these sorts of tutorials Monoid comes first, its the simplest, easiest most familiar sort of thing, we don't really even need code to describe it, its loosely the theory of having something you can composing two things together and getting another thing of the same type, together with an identity element, so in mathematics the integers form a monoid with `+` and the identity element is `0` or we can use multiplication `*` and `1`, the set of booleans, i.e. `{ True , False }` together with the operator `AND` form a monoid, where the identity element is `True`, to be honest, these things are rarely used as monoids in functional code, they could be, there just isn't much use, the things that are usually used as monoids are collections, so in the XPath world this is `sequence`, `array`, `map` (and the recently added `maybe`).

So lets be a bit more formal. we need a function, we're going to call it `append` (which is a bit of a clue!) with pseudo type:

`function(A,A) as A`

and we need this function to satisfy these rules:

* its associative i.e.
  * `append(append($a,$b),$c) == append($a,append($b,$c))`
* there exists an element `zero` such that
  * for any element x, `append($zero,$x) == $x`
  * for any element x, `append($x,$zero) == $x`

The name `append` is a bit of a clue as to what our intentions are and this is the usual name (in functional languages, or some variation, though in mathematics less so).

**4.1 Sequence monoid?**

I must admit at this point, to being scared, I relegated Monoid to further down the list, not because Monoids are hard, but because XPath is, despite doing XSLT for many years the actual behaviour of sequences I find at times counter intuitive, and this blog is really about highlighting the idiosyncrasies.

First let me just clarify `==`, this doesn't just means there is some operator equals defined that returns true, it means that in every program in every context, I can replace the expression on the left with the one on the right (and vice versa) and it will be indistinguishable without looking at the code. Some of the behaviour of XPath makes that quite difficult to be sure.

Lets consider the things I am (slightly) more confident of, for `array`, I'm pretty sure I can write `append` for `array`, where I'm confident about arrays is their order is fixed, so I'm confident if I append an array to another array, I'm going to get the expected order. I can visualise the empty array as the identity element and that seems to work.

Maybe monoids are a bit quirky, lets leave them for a bit.

Maps, well they're a bit harder, but lets leave those to last, the piece that worries me, it **always** worries me, is sequences. Sequences do all sort of weird things to ordering that frankly I don't understand, and even when I do, the complexities quickly multiply in my head to confuse me, so my standard approach is to make as few assumptions as possible about ordering, and impose my own explicit order where I can.

Lets take the bull by the horns though, and see what we can do with sequence, here are some concrete examples of how sequences behave:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <xsl:template match="/">
        <xsl:variable name="doc" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                </root>
            </xsl:document>
        </xsl:variable>
        <output>
            <xsl:sequence select="$doc/root/bar | $doc/root/foo"/>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

what does this return........

```xml
<output>
   <foo/>
   <bar/>
</output>
```

oooo, I was half expecting this, so it returns `foo` first because `|` imposes document order, and foo comes before bar in my document definition, interesting.

how about this.

```xml
<output>
    <xsl:sequence select="$doc/root/bar | $doc/root/foo | $doc/root/bar"/>
</output>
```

any ideas (I knew this one!)

actually its the same as before, it removes duplicates and puts things in document order.

what about this one....I'm really genuinely not sure, if `|` imposes document order, then what happens with two document?

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <xsl:template match="/">
        <xsl:variable name="doc1" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                </root>
            </xsl:document>
        </xsl:variable>
        <xsl:variable name="doc2" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                </root>
            </xsl:document>
        </xsl:variable>
        <output>
            <xsl:sequence select="$doc1/root/bar | $doc2/root/foo | $doc1/root/bar"/>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

what's document order now? (I've not got a clue)

```xml
<output>
   <bar/>
   <foo/>
</output>
```

ok, well, presumably its document order but based on the order of the documents inside the sequence? (this is bad feeling though, I can't find the definition in the spec, and just because I think I know what the environment is doing, I'm guessing, and it may change unless the specification says it can't), and the notion of order seems at best complex.

Ok, but think I CAN construct sequences with duplicates, consider:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <xsl:template match="/">
        <xsl:variable name="doc" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                </root>
            </xsl:document>
        </xsl:variable>
        <output>
            <xsl:sequence select="($doc/root/bar,$doc/root/bar)"/>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

this results in

```xml
<output>
   <bar/>
   <bar/>
</output>
```

Ok, so sequences DO support duplicated but it was the `|` operator removes them?

Lets do an experiment:

```xml
<xsl:sequence select="($doc/root/bar,$doc/root/bar) | ()"/>
```

and we get

```xml
<output>
   <bar/>
</output>
```

ah.....that's not good.

`()` is the obvious candidate for our identity element, but we've just demonstrated that it doesn't work, the operator `|` forced duplicates to be removed from the resulting sequence.

So we cannot construct a monoid with a `sequence` and `|` . I think that's a shame, document order is a big thing in xpath, but I do think the above examples show how the language is unintuitive and its challenging (impossible for me) to build a simple mental model of what a sequence is.

All is not lost though, we have stumbled on a new candidate append though and `(,)` allowed us to create a sequence with duplicates, so lets try that, here's a demonstration of it, and an example of it obeying the rules:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <xsl:template match="/">
        <xsl:variable name="doc" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                    <wibble/>
                </root>
            </xsl:document>
        </xsl:variable>
        <output>
            <associative>
                <expression1>
                    <xsl:sequence select="(($doc/root/wibble),($doc/root/bar,$doc/root/foo))"/>
                </expression1>
                <expression2>
                    <xsl:sequence select="(($doc/root/wibble,$doc/root/bar),($doc/root/foo))"/>
                </expression2>
            </associative>
            <identity>
                <expression1>
                    <xsl:sequence select="((),($doc/root/wibble,$doc/root/bar))"/>
                </expression1>
                <expression2>
                    <xsl:sequence select="(($doc/root/wibble,$doc/root/bar),())"/>
                </expression2>
            </identity>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

OK, this isn't a proof but, it gives some indication, and the results are:

```xml
<output>
   <associative>
      <expression1>
         <wibble/>
         <bar/>
         <foo/>
      </expression1>
      <expression2>
         <wibble/>
         <bar/>
         <foo/>
      </expression2>
   </associative>
   <identity>
      <expression1>
         <wibble/>
         <bar/>
      </expression1>
      <expression2>
         <wibble/>
         <bar/>
      </expression2>
   </identity>
</output>
```

So I think we CAN make `sequence` and `(,)` into a monoid! (let me know if you think this is wrong!).

and we can write the above code like this:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                xmlns:xsequence="www.kookerella.com/xsequence"
                version="3.0">
    <xsl:output method="xml" indent="yes"/>

    <!-- empty as sequence A -->
    <!-- empty : Sequence A -->
    <xsl:function name="xsequence:empty" as="item()*">
        <xsl:sequence select="()"/>
    </xsl:function>

    <!-- append as function(sequence A, sequence A) as sequence A -->
    <!-- append : (Sequence A,Sequence A) -> Sequence A -->
    <xsl:function name="xsequence:append" as="item()*">
        <xsl:param name="sequence1" as="item()*"/>
        <xsl:param name="sequence2" as="item()*"/>
        <xsl:sequence select="$sequence1,$sequence2"/>
    </xsl:function>

    <xsl:template match="/">
        <xsl:variable name="doc" as="document-node()">
            <xsl:document>
                <root>
                    <foo/>
                    <bar/>
                    <wibble/>
                </root>
            </xsl:document>
        </xsl:variable>
        <output>
            <associative>
                <expression1>
                    <xsl:sequence select="xsequence:append(($doc/root/wibble),($doc/root/bar,$doc/root/foo))"/>
                </expression1>
                <expression2>
                    <xsl:sequence select="xsequence:append(($doc/root/wibble,$doc/root/bar),($doc/root/foo))"/>
                </expression2>
            </associative>
            <identity>
                <expression1>
                    <xsl:sequence select="xsequence:append(xsequence:empty(),($doc/root/wibble,$doc/root/bar))"/>
                </expression1>
                <expression2>
                    <xsl:sequence select="xsequence:append(($doc/root/wibble,$doc/root/bar),xsequence:empty())"/>
                </expression2>
            </identity>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

Note, I've used the names `empty` and `append`, and note the comments for the types of these functions.

**4.2 Array monoid**

Array, we'll skip, the code can be found in the git repository, its simple and doesn't really cover anything we haven't covered above, array is quite well behaved.

**4.3 Map monoid**

Maps are a bit more tricky, the obvious answers are that `append` can be defined in terms of `map:merge` and `empty` is `map {}`, there needs to be a bit of care over ensuring that `append` is associative, and the case where there are duplicate entries i.e. the same key exists in two maps.

`map:merge` has several duplicate-key options: `reject` raises an error on duplicate keys, `use-first`/`use-last` keep the first or last value for a key, `use-any` keeps an implementation-defined one, and `combine` concatenates all the values for a key into a sequence.

So for brevity lets just use `++` to indicate append, and consider this expression:

`(a ++ b) ++ c` vs `a ++ (b ++ c)`

and we need to be sure that any program with the expression on the left would be unchanged if we replaced it with the expression on the right.

If there are no duplicated keys, then I'm happy it doesn't matter in which order you merge the maps (we probably need to be a little careful about ordering, but I'm personally willing to ignore ordering as I think developers wont be depending on specific orders when running executing code via `map:for-each` etc).

If there are duplicated keys then we need to think about which options we can use.

`use-any` we can reject, if we can't depend on which keys are kept and discarded then we can't safely substitute the expressions (and in fact we can't safely run the same code twice and be sure of the outcome, even though the code oddly claims the function is deterministic).

`use-first` works.

`use-last` works.

`combine` is odd, because it can change the type of the resulting map, i.e. we can take two `map(xs:string,xs:string)` and the result is a `map(xs:string,xs:string*)`.

To be honest, I find `combine` to be the most intuitive behaviour and as all maps of type `map(xs:string,xs:string)` are satisfy the type `map(xs:string,xs:string*)` I'm willing to also turn a blind eye.

So you can create three different monoids, I'll discuss sensible ways to do that later, but for the moment we'll stick with using `combine` and implement it like this:

```xml
<xsl:function name="xmap:empty" as="map(*)">
    <xsl:sequence select="map {}"/>
</xsl:function>

<xsl:function name="xmap:append" as="map(*)">
    <xsl:param name="map1" as="map(*)"/>
    <xsl:param name="map2" as="map(*)"/>
    <xsl:sequence select="map:merge(($map1,$map2),map { 'duplicates' : 'combine' })"/>
</xsl:function>
```

In order to test this we have a minor problem, we can't serialise maps to xml, and if we serialise to json, we can't serialise sequences of values, our values need to be held in arrays....

....but, we now have quite a sophicated set of tools (for once I'm not going to post all the code, but do by 'including' the relevant code libraries, but here is our experimental test case:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xmap="http://kookerella.com/xmap"
                xmlns:map="http://www.w3.org/2005/xpath-functions/map"
                xmlns:xarray="http://kookerella.com/xarray"
                xmlns:sequence="http://kookerella.com/xsl:sequence">
    <xsl:output method="json"/>
    <xsl:include href="SimpleNamespace/sequence.xslt"/>
    <xsl:include href="SimpleNamespace/xarray.xslt"/>
    <xsl:include href="SimpleNamespace/xmap.xslt"/>

    <xsl:template match="/">
        <xsl:sequence select="
            xmap:mapValue(
                sequence:toArray#1,
                xmap:append(
                    map { '1' : '2'},
                    map { '1' : '2' }))"/>
    </xsl:template>
</xsl:stylesheet>
```

We're testing, `xmap:append` really, but we have to `map` the key values to arrays in order to serialise them, but we get this result:

```json
{"1":["2","2"]}
```

and so I'm happy that this is a sensible monoid implementation.

**4.4 Maybe monoid**

This is where it becomes interesting; my initial thought is that 'Maybe' resembles an array limited to a maximum size of one. So, the question arises: how can I concatenate two arrays of size one and still end up with an array of size one? It seems unfeasible, doesn't it? After all, 1+1 equals 2, surely? Well sometimes it does work!.

If the thing, lets call it A, inside the 'Maybe' is a monoid, then we can make 'Maybe A' into a monoid!....

We haven't yet defined the mechanism to do this in code, so at this stage we can only talk about it.

So I claim that a `Maybe (Sequence xs:integer)` can be made into a monoid because `Sequence xs:integer` is a monoid.

Lets just demonstrate this by writing some examples (the symbol here `->` simply means evaluates to, it isn't part of XPath etc):

`maybe:append(maybe:none(),maybe:none()) -> maybe:none()`

ok, so that's intuitive, we append two `none`s we get a `none`.

`maybe:append(maybe:some((1,2,3)),maybe:none()) -> maybe:some((1,2,3))`

i.e. if we have a `some` and a `none`, we just take the `some`.

and similarly:

`maybe:append(maybe:none(),maybe:some((1,2,3))) -> maybe:some((1,2,3))`

i.e. if we have a `none` and a `some` we take the `some`.

now the tricky one, how do we make 1+1=1.

`maybe:append(maybe:some((4,5,6)),maybe:some((1,2,3))) -> maybe:some(sequence:append((4,5,6),(1,2,3))))`

!...because `sequence` is a monoid, then we can append the contents of the `some`s

and we get

`....-> maybe:some((4,5,6,1,2,3))))`

is it associative? yes, because the monoid we defined on sequence is associative, if both parameters are `some`, and the values over which we are `append`ing support `append`, the we can move the `append` into the inner type space.

do we have a left and right identity/empty element? yes, we have 2! both `maybe:none()` and `maybe:some(sequence:none())` (we didn't say we couldn't have more than 1, we just need to pick one).

Later we will see how to construct the code for this, but at the moment we are constrained by our simplified setup, and we just need to be content that in theory at least we can sometimes construct monoids from maybe.
