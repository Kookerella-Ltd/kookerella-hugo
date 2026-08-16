---
title: "Functional XSLT/XPath/XQuery - #3 Maybe data type"
date: 2024-06-05
slug: "functional-xsltxpathxquery-3-maybe-data-type"
tags: ["xslt", "xpath", "xquery", "functional-programming"]
---

# Functional XSLT/XPath/XQuery - #3 Maybe data type

We are now in a position to fill in a missing data type.

Lets cover some XPath basics.

Consider the following XSLT

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:kooks="http://kookerella.com">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="kooks:safeDivision" as="xs:numeric?">
        <xsl:param name="numerator" as="xs:numeric"/>  
        <xsl:param name="denominator" as="xs:numeric"/>
        <xsl:if test="not($denominator eq 0)">
            <xsl:sequence select="$denominator div $denominator"/>
        </xsl:if>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:variable
                name="calculations"
                as="xs:numeric*"
                select="(1 div 1,1 div 2,2 div 3)"/>
            <xsl:sequence select="$calculations"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

If we execute it against an xml it returns this:

```xml
<output>1 0.5 0.666666666666666667</output>
```

Fair enough, we have a list of three calculations, but this is contrived, maybe in our production code they're not hard coded like the above but come from the input file, and that input file can create data that will inevitably end with a possible division by zero, if we were to hard code this scenario it would be like this:

```xml
<xsl:variable name="calculations" as="xs:numeric*"
     select="(
        1 div 1,
        1 div 2,
        2 div 0)"/>
```

If we run this it fails with `Integer division by zero`, that's what we expected, but we wouldn't want this to happen in production code for valid data.

Hmmmm, lets try writing our own division, lets try to only return a result if the calculation is valid:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:kooks="http://kookerella.com">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="kooks:safeDivision" as="xs:numeric?">
        <xsl:param name="numerator" as="xs:numeric"/>  
        <xsl:param name="denominator" as="xs:numeric"/>
        <xsl:if test="not($denominator eq 0)">
            <xsl:sequence select="$numerator div $denominator"/>
        </xsl:if>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <xsl:variable name="calculations" as="xs:numeric*"
                select="(
                    kooks:safeDivision(1,1),
                    kooks:safeDivision(1,2),
                    kooks:safeDivision(2,0))"/>
            <xsl:sequence select="$calculations"/>
        </output>
    </xsl:template>
</xsl:stylesheet>
```

This result in:

```xml
<output>1 0.5</output>
```

Success!?

Well yes and no, the invalid result has been supressed, but its as if it didn't happen, its silently disappeared, lets look at the types here, `kooks:safeDivision` returns an `xs:numeric?` a sequence thats constrained to be of length 0 or 1. Our sequence of calculations is of type `xs:numeric*`, so the notion of the the calculation returning an optional answer is lost, you would expect the type to be something like `xs:numeric?*` i.e. sequence of optional results, but this type doesn't exist, there is some magic happening that makes this type effectively useless even if it did exist.

If we count the number of elements in `$calculations` it would be 2, not 3, even though there are 3 calculations there are only 2 results, something surprising is happening here, that isn't just confined to sequences constrained by `?`.

Consider this simple example:

```xml
<xsl:template match="/">
    <output>
        <xsl:sequence select="count((1,2,(3,8,(10,20)),4,5))"/>
    </output>
</xsl:template>
```

The nesting lists or arrays is sometimes used as a simple mechanism for modelling trees (and forests), when I first tripped on this I was very confused, I saw the expression above and thought the elements in the sequence were `1`,`2`,`(3,8,(10,20))`,`4`,`5` i.e. there are 5 elements, the fact that the 3rd element is a sequence seemed to me to be irrelevant, but XSLT says the answer is:

```xml
<output>8</output>
```

So what is happening? I then discovered that XPath, doesn't support nested sequences, but it doesn't throw an error, it auto flattens the list, the code is equivalent to:

```xml
<xsl:sequence select="count((1,2,3,8,10,20,4,5))"/>
```

Again, like the other idiosyncrasies, the specifiers of XPath decided this was desirable behaviour and its now boiled into the language.

Subsequently the `array` data type has been introduced and nesting arrays does behave as you would expect, so this:

```xml
<xsl:sequence select="
    array:size(
        array {
            1,
            2,
            array {
                3,
                8,
                array {
                    10,
                    20
                }
            },
            4,
            5
       })"/>
```

results in:

```xml
<output>5</output>
```

So we can model nested sequences as arrays instead of sequences (i.e. `*`) and the nesting is preserved, but how can we apply this to our 'safe' division? We could just model optional sequences (i.e. `?`) using arrays, and use our `xarray:map` function defined in the previous blog to use it, and all the existing functions defined in the array namespace as well.

Well yes, we could, but there are things we would quite like to do with optional values that don't apply to arrays e.g. maybe a default function would be useful for optional values, and this wouldn't really make sense to arrays in general and just clutter the existing box of tools for array, we also actually want our code to fail if someone puts two values inside something that should only contain, at most, one, for sequences we can do this by annotating our types with `?` but we cant do the same thing with an array.

It probably makes sense we define a new type.

Defining new types in XSLT is problematic, we basically have these options:

* we define it in terms of XML, XSLT in particular is very proficient in allowing us to define functions that create and process XML.
* we define it in terms of 2 Maps that simulate a value with a none value.
* we define it in terms of an array, technically the array will allow more than one value, but we can define functions to enforce this where possible, and assume that's the case elsewhere.
* we could use a fancy final encoding.....but lets leave that for a different blog.

There are pros and cons to each of these approaches, we will choose to model using array based principally on simplicity of implementation (and explanation).

Lets step through the code (and present a complete executable XSLT at the end):

First we need to define functions that capture the concept of there being a value or not a value i.e **Maybe** a value.

```xml
<xsl:function name="maybe:some" as="array(*)">
    <xsl:param name="value" as="item()*"/>
    <xsl:sequence select="array { $value }"/>
</xsl:function>

<xsl:function name="maybe:none" as="array(*)">
    <xsl:sequence select="array {}"/>
</xsl:function>
```

Here we can see that a `none` value is an empty array, and a `some` value is an array of size 1.

Lets can implement the functor pattern:

```xml
<xsl:function name="maybe:map" as="array(*)">
    <xsl:param name="mapper" as="function(item()*) as item()*"/>
    <xsl:param name="maybe" as="array(*)"/>
    <xsl:sequence select="array:for-each($maybe,function($a) { $mapper($a) })"/>  
</xsl:function>
```

(This is the identical implementation as we saw for `xarray:map`)

We've created a mechanism to `match` the two cases to simplifies our logic:

```xml
<!-- maybe:match as function(function() as B,function(A) as B, Maybe A) as B -->
<!-- maybe:match :: ((() -> B),(A -> B),Maybe A) -> B -->
<xsl:function name="maybe:match" as="item()*">
    <xsl:param name="visitNone" as="function() as item()*"/>
    <xsl:param name="visitSome" as="function(item()*) as item()*"/>
    <xsl:param name="maybe" as="array(*)"/>
    <xsl:choose>
        <xsl:when test="array:size($maybe) = 0">
            <xsl:sequence select="$visitNone()"/>
        </xsl:when>
        <xsl:otherwise>
            <xsl:sequence select="$visitSome(array:get($maybe,1))"/>
        </xsl:otherwise>
    </xsl:choose>
</xsl:function>
```

(take note the type of the match function in the comment, its good to use the type as a summary of what the function does)

and we can subsequently use this to create a utility function to allow us to serialise our data to a string.

```xml
<xsl:function name="maybe:pprint" as="xs:string">
    <xsl:param name="maybe" as="array(*)"/>
    <xsl:sequence select="
        maybe:match(
            function() { 'None' },
            function($value) { 'Some(' || $value || ')' },
            $maybe)"/>
</xsl:function>
```

We can implement our safe division function like this:

```xml
<xsl:function name="kooks:safeDivision" as="array(xs:numeric)">
    <xsl:param name="numerator" as="xs:numeric"/>  
    <xsl:param name="denominator" as="xs:numeric"/>
    <xsl:choose>
        <xsl:when test="not($denominator eq 0)">
            <xsl:sequence select="maybe:some($numerator div $denominator)"/>
        </xsl:when>
        <xsl:otherwise>
            <xsl:sequence select="maybe:none()"/>
        </xsl:otherwise>
    </xsl:choose>
</xsl:function>
```

So we can see here that we explicitly return `none` where before we tried to simply return an empty sequence that was then flattened into the calculation sequence and effectively erased.

Putting it all together:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:array="http://www.w3.org/2005/xpath-functions/array"
                xmlns:maybe="http://www.kookerella.com/maybe"
                xmlns:sequence="http://kookerella.com/xsl:sequence"
                xmlns:kooks="http://www.kookerella.com">
    <xsl:output method="xml" indent="yes"/>

    <xsl:function name="kooks:safeDivision" as="array(xs:numeric)">
        <xsl:param name="numerator" as="xs:numeric"/>  
        <xsl:param name="denominator" as="xs:numeric"/>
        <xsl:choose>
            <xsl:when test="not($denominator eq 0)">
                <xsl:sequence select="maybe:some($numerator div $denominator)"/>
            </xsl:when>
            <xsl:otherwise>
                <xsl:sequence select="maybe:none()"/>
            </xsl:otherwise>
        </xsl:choose>
    </xsl:function>

    <xsl:function name="sequence:map" as="item()*">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="sequence" as="item()*"/>
        <xsl:sequence select="$sequence ! $mapper(.)"/>
    </xsl:function>

    <xsl:function name="maybe:some" as="array(*)">
        <xsl:param name="value" as="item()*"/>
        <xsl:sequence select="array { $value }"/>
    </xsl:function>

    <xsl:function name="maybe:none" as="array(*)">
        <xsl:sequence select="array {}"/>
    </xsl:function>

    <xsl:function name="maybe:map" as="array(*)">
        <xsl:param name="mapper" as="function(item()*) as item()*"/>
        <xsl:param name="maybe" as="array(*)"/>
        <xsl:sequence select="array:for-each($maybe,function($a) { $mapper($a) })"/>  
    </xsl:function>

    <xsl:function name="maybe:match" as="item()*">
        <xsl:param name="visitNone" as="function() as item()*"/>
        <xsl:param name="visitSome" as="function(item()*) as item()*"/>
        <xsl:param name="maybe" as="array(*)"/>
        <xsl:choose>
            <xsl:when test="array:size($maybe) = 0">
                <xsl:sequence select="$visitNone()"/>
            </xsl:when>
            <xsl:otherwise>
                <xsl:sequence select="$visitSome(array:get($maybe,1))"/>
            </xsl:otherwise>
        </xsl:choose>
    </xsl:function>

    <xsl:function name="maybe:pprint" as="xs:string">
        <xsl:param name="maybe" as="array(*)"/>
        <xsl:sequence select="
            maybe:match(
                function() { 'None' },
                function($value) { 'Some(' || $value || ')' },
                $maybe)"/>
    </xsl:function>

    <xsl:template match="/">
        <output>
            <!-- calculations as maybe(xs:numeric)* -->
            <xsl:variable name="calculations" as="array(*)*"
                select="(
                    kooks:safeDivision(1,1),
                    kooks:safeDivision(1,2),
                    kooks:safeDivision(2,0))"/>
            <xsl:sequence select="sequence:map(maybe:pprint#1, $calculations)"/>
        </output>  
    </xsl:template>
</xsl:stylesheet>
```

Note we've used `sequence:map` here, i.e. we're starting to utilise combinations of our new tooling:

If we execute this against an XML we get:

```xml
<output>Some(1) Some(0.5) None</output>
```

And we see 3 explicit results. We can now model optional values explicitly, and even sequences of them, we can print them, match them and map them.

Note - the use of `array` for the calculations, we should really think of the type of the variable calculations as `maybe(xs:numeric)*` i.e. a sequence of optional values. (I believe you can tidy this up using type aliases in saxon, but I've never tried it, and it would make the code saxon specific).

Next time we will introduce more idioms and implement them in all out data types including Maybe.
