---
title: "Functional XSLT/XPath/XQuery - #5 Monad pattern"
date: 2024-06-15
slug: "functional-xsltxpathxquery-5-monad-pattern"
tags: ["xslt", "xpath", "xquery", "functional-programming", "monad"]
---

# Functional XSLT/XPath/XQuery - #5 Monad pattern

The monad is probably the most written about functional pattern, it is famous for convoluted and obtuse explanations, but I will steal some of the better explanations and translate them into XSLT/XPath.

(adapted directly from [All About Monads - HaskellWiki](https://wiki.haskell.org/All_About_Monads)).

"Suppose that we are writing a program to keep track of sheep cloning experiments. We would certainly want to know the genetic history of all of our sheep, so we would need `mother` and `father` functions. But since these are cloned sheep, they may not always have both a mother and a father!

We would represent the possibility of not having a mother or father using the `Maybe` type"

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:xmap="http://kookerella.com/xmap"
                xmlns:map="http://www.w3.org/2005/xpath-functions/map"
                xmlns:xarray="http://kookerella.com/xarray"
                xmlns:kooks="http://kookerella.com"
                xmlns:maybe="http://www.kookerella.com/maybe"
                xmlns:array="http://www.w3.org/2005/xpath-functions/array"
                xmlns:sequence="http://kookerella.com/xsl:sequence">
    <xsl:output method="json"/>
    <xsl:include href="SimpleNamespace/maybe.xslt"/>

    <xsl:function name="kooks:makeSheep" as="map(*)">
        <xsl:param name="name" as="xs:string"/>
        <xsl:param name="fatherMaybe" as="array(map(*))"/>
        <xsl:param name="motherMaybe" as="array(map(*))"/>
        <xsl:sequence select="
            map {
                'name' : $name,
                'fatherFather' : $fatherMaybe,
                'motherMaybe' : $motherMaybe
            }"/>
    </xsl:function>

    <xsl:function name="kooks:getFatherMaybe" as="array(map(*))">
        <xsl:param name="sheep" as="map(*)"/>
        <xsl:sequence select="map:get($sheep,'fatherMaybe')"/>
    </xsl:function>

    <xsl:function name="kooks:getMotherMaybe" as="array(map(*))">
        <xsl:param name="sheep" as="map(*)"/>
        <xsl:sequence select="map:get($sheep,'motherMaybe')"/>
    </xsl:function>

    <xsl:template match="/">
        <xsl:variable name="jim" as="map(*)"
            select="kooks:makeSheep('jim',maybe:none(),maybe:none())"/>
        <xsl:variable name="dawn" as="map(*)"
            select="kooks:makeSheep('dawn',maybe:none(),maybe:none())"/>
        <xsl:variable name="dolly" as="map(*)"
            select="kooks:makeSheep('dolly',maybe:some($dawn),maybe:none())"/>
        <xsl:sequence
            select="map { 'result' : kooks:getMotherMaybe($dolly) }"/>
    </xsl:template>
</xsl:stylesheet>
```

Here we can create sheep with no father and/or no mother (I've not given jim and dawn parents, I have to stop somewhere, we will assume divine intervention, these are the Adam and Eve of our sheep population, and dolly is a clone of dawn).

lets try and define functions to get `maternalGrandfather` and `mothersPaternalGrandfather` like this:

```xml
<xsl:function name="kooks:getMaternalGrandfatherMaybe" as="array(map(*))">
    <xsl:param name="sheep" as="map(*)"/>
    <xsl:sequence select="
        maybe:match(
            function() {
                maybe:none()
            },
            function($mother) {
                kooks:getFatherMaybe($mother)
            },
            kooks:getMotherMaybe($sheep)"/>
</xsl:function>
```

Its a little convoluted, but we try to get the mother, and then match, if its none, we return none, and if it isn't, we try to get the father.

Lets try something harder:

```xml
<xsl:function name="kooks:getMothersPaternalGrandfatherMaybe" as="array(map(*))">
    <xsl:param name="sheep" as="map(*)"/>
    <xsl:sequence select="
        maybe:match(
            function() {
                maybe:none()
            },
            function($mother) {
                maybe:match(
                    function() {
                        maybe:none()
                    },
                    function($gfather) {
                        kooks:getFatherMaybe($gfather)
                    },
                    $mother
                ),
                kooks:getFatherMaybe($mother)
        },
        $sheep)"/>  
</xsl:function>
```

that's not nice, we can see that this is getting out of hand, every time we look another generation back we get a new match.

But notice there is a repeated pattern, we match a maybe, and map `none` to `none`.

We can extract out that repeated pattern like this, and effectively pass in a function that captures what we need to do when we match `some`:

```xml
<!-- maybe:bind as function(function(A,maybe(B)),maybe(A)) as maybe(B)  -->
<!-- maybe:bind :: (A -> maybe(B),maybe(A)) -> maybe(B) -->
<xsl:function name="maybe:bind" as="array(*)">
    <xsl:param name="binder" as="function(item()*) as array(*) "/>
    <xsl:param name="maybe" as="array(*)"/>
    <xsl:sequence select="
        maybe:match(
            function() {  
                maybe:none()
            },
            function($value) {
                $binder($value)
            },
            $maybe)"/>
</xsl:function>
```

then we can simplify the code, to this:

```xml
<xsl:function name="kooks:getMothersPaternalGrandfatherMaybe" as="array(map(*))">
    <xsl:param name="sheep" as="map(*)"/>
    <xsl:sequence select="
        maybe:bind(
            kooks:getFatherMaybe#1,
            maybe:bind(
                kooks:getFatherMaybe#1,
                maybe:bind(
                    kooks:getMotherMaybe#1,
                    maybe:some($sheep))))"/>
</xsl:function>
```

to be fair that's much better, I can in fact just read the sequence of calls almost as if they were a sequence of nested calls, 1st we get the mother, then the father of the mother and then the father of the father of the mother.

Similarly, monads are useful with nested arrays (basically nested loops):

Writing bind over an array is not completely trivial, and you can skip the details if you want. First we create a concat function that takes an array of arrays, and concatenates them. Given that we can build bind by mapping over the binder function and concatenating the result (this may not be the most efficient implementation).

```xml
<xsl:function name="xarray:concat" as="array(*)">
    <xsl:param name="arrayOfArrays" as="array(array(*))"/>
    <xsl:sequence select="
        array:fold-left(
            $arrayOfArrays,
            [],
            xarray:append#2)"/>
</xsl:function>

<!-- array:map as function(function(A) as array(B),array(A)) as array(B) -->
<!-- array:map :: ((A -> array B),array A) -> array B -->
<xsl:function name="xarray:bind" as="array(item()*)">
    <xsl:param name="mapper" as="function(item()*) as array(item()*)"/>
    <xsl:param name="array" as="array(item()*)"/>
    <xsl:sequence select="
        xarray:concat(
            array:for-each(
                $array,
                $mapper))"/>
</xsl:function>
```

Now we can demonstrate the usage of bind over a contrived example of querying visitors to a blog:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                exclude-result-prefixes="#all"
                version="3.0"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:xmap="http://kookerella.com/xmap"
                xmlns:map="http://www.w3.org/2005/xpath-functions/map"
                xmlns:xarray="http://kookerella.com/xarray"
                xmlns:kooks="http://kookerella.com"
                xmlns:maybe="http://www.kookerella.com/maybe"
                xmlns:array="http://www.w3.org/2005/xpath-functions/array"
                xmlns:sequence="http://kookerella.com/xsl:sequence">
    <xsl:output method="json"/>
    <xsl:include href="SimpleNamespace/xmap.xslt"/>
    <xsl:include href="SimpleNamespace/xarray.xslt"/>

    <xsl:function name="kooks:makeBlog" as="map(*)">
        <xsl:param name="title" as="xs:string"/>
        <xsl:param name="visitors" as="array(xs:string)"/>
        <xsl:sequence select="
            map {
                'title' : $title,
                'visitors' : $visitors
            }"/>
    </xsl:function>

    <xsl:function name="kooks:getVisitors">
        <xsl:param name="blog" as="map(*)"/>
        <xsl:sequence select="map:get($blog,'visitors')"/>
    </xsl:function>

    <xsl:function name="kooks:makeWebsite" as="map(*)">
        <xsl:param name="url" as="xs:string"/>
        <xsl:param name="blogs" as="array(map(*))"/>
        <xsl:sequence select="
            map {
                'url' : $url,
                'blogs' : $blogs
            }"/>
    </xsl:function>

    <xsl:function name="kooks:getBlogs">
        <xsl:param name="website" as="map(*)"/>
        <xsl:sequence select="map:get($website,'blogs')"/>
    </xsl:function>

    <xsl:template match="/">
        <xsl:variable name="websites"
            select="[
                kooks:makeWebsite(
                    'kookerella.com',
                    [
                        kooks:makeBlog(
                            'functional xslt',
                            [
                                'Mr Blogs',
                                'Miss Smith'
                            ]
                        )
                    ]
                ),
                kooks:makeWebsite(
                    'acme.com',
                    [
                        kooks:makeBlog(
                            'fun with power tools',
                            [
                                'Miss Smith',
                                'Eugene'
                            ]
                        )
                    ]
                )
            ]"/>
        <xsl:sequence select="
            map {
                'result' :
                    xarray:bind(
                        kooks:getVisitors#1,
                        xarray:bind(
                            kooks:getBlogs#1,
                            $websites))
            }"/>
    </xsl:template>
</xsl:stylesheet>
```

but note, the structure of the final query is very similar to the maybe example, first we get the blogs, then we get the visitors, the structure of the code is identical the only difference is the underling data type.

Note we can write this in a slightly more convoluted manner like this (I've inserted a map):

```xml
<xsl:sequence select="
    map {
        'result' :
            xarray:map(
                function:id#1,
                xarray:bind(
                    kooks:getVisitors#1,
                    xarray:bind(
                        kooks:getBlogs#1,
                        $websites)))
    }"/>
```

and consider this query:

`for $website in $websites,`
`$blog in kooks:getBlogs($website),`
`$visitor in kooks:getVisitors($blog)`
`function:id($visitor)`

and you should note the structure is very similar (its just upside down, and the variables are explicit).

This pattern is the dna of for expressions in xpath and xquery, in fact its the dna of all these sorts of looping construct in any language, but as we've seen above its the same pattern for maybe, so anything that we can implement a monad on, can, in theory at least, be written using this `for` expression style. This fact is used in languages like C# for LINQ, Haskell for 'do notation' or F# for computational expressions to allow developers to write their own monadic constructs and then write expressions in a 'query' style language (Philip Wadler was instrumental in creating 'do notation' and also the XQuery language, the same theory underpins both).

We could go on, there are a whole raft of things we can turn into Monads (as there were with Monoids and Functors, but these we will leave to other blogs, you can see them in the github repository).
