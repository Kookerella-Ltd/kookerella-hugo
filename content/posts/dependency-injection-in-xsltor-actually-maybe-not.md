---
title: "Dependency injection in XSLT...or actually maybe not"
date: 2024-05-08
slug: "dependency-injection-in-xsltor-actually-maybe-not"
tags: ["xslt", "dependency-injection", "interpreter"]
---

# Dependency injection in XSLT...or actually maybe not

Ever written two, three or more stylesheets that are almost the same, but not quite the same, maybe you've struggled with #import or #include to separate your core shared code from the bespoke specific code? Maybe you're an OO programmer steeped in the theology of objects, favouring composition over inheritance and find XSLT innate mechanisms to close to its OO cousins? Or a functional programmer at ease with higher order functions, but dabbling in some XSLT and wondering how to apply you functional skills, well read on, hopefully this blog will help you apply these skills in XSLT but ironically question whether these techniques really are the "state of the art", many believe and when you return to your Scala/C#/Java/C++ or whatever it is you do most of the time, question whether there is more than one way to skin the cat (ew...I hate that phrase).

Lets make it real, lets say we have a requirement to write two stylesheets to transforms some xml data, here is the input:

```xml
<library>
    <publisher name="Wiley Group">
        <book>
            <title>The Green Mile</title>
            <author>Stephen King</author>
            <price>18</price>
        </book>
        <book>
            <title>The Catcher in the Rye</title>
            <author>J. D. Salinger</author>
            <price>25</price>
        </book>
        <book>
            <title>I, Robot</title>
            <author>Issac Asimov</author>
            <price>15</price>
        </book>
    </publisher>
    <publisher name="Time Publishing">
        <book>
            <title>Foundation Novels</title>
            <author>Isaac Asimov</author>
            <price>30</price>
        </book>
        <book>
            <title>Pygmalion</title>
            <author>Oscar Wilde</author>
            <price>10</price>
        </book>
    </publisher>
    <publisher name="Amazon">
        <book>
            <title>Dune</title>
            <author>Frank Herbert</author>
            <price>25</price>
        </book>
        <book>
            <title>Dune Messiah</title>
            <author>Frank Herbert</author>
            <price>12</price>
        </book>
    </publisher>
</library>
```

And we need two output formats, flavourA and flavourB, here is flavourA:

```xml
<FlavourA>
    <book publisher="Wiley Group">
        <title>The Green Mile</title>
        <author>Stephen King</author>
        <price>18</price>
    </book>
    <book publisher="Wiley Group">
        <title>The Catcher in the Rye</title>
        <author>J. D. Salinger</author>
        <price>25</price>
    </book>
    <book publisher="Wiley Group">
        <title>I, Robot</title>
        <author>Issac Asimov</author>
        <price>15</price>
    </book>
    <book publisher="Time Publishing">
        <title>Foundation Novels</title>
        <author>Isaac Asimov</author>
        <price>30</price>
    </book>
    <book publisher="Time Publishing">
        <title>Pygmalion</title>
        <author>Oscar Wilde</author>
        <price>10</price>
    </book>
    <book publisher="Amazon">
        <title>Dune</title>
        <author>Frank Herbert</author>
        <price>25</price>
    </book>
    <book publisher="Amazon">
        <title>Dune Messiah</title>
        <author>Frank Herbert</author>
        <price>12</price>
    </book>
</FlavourA>
```

To generate it, we write the following xslt:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <FlavourA>
            <xsl:for-each select="library/publisher">
                <xsl:variable name="publisher" select="." as="element(publisher)"/>
                <xsl:for-each select="book">
                    <book>
                        <publisher>{$publisher/@name}</publisher>
                        <title>{title/text()}</title>
                        <author>{author/text()}</author>
                        <price>{price/text()}</price>
                    </book>
                </xsl:for-each>  
            </xsl:for-each>
        </FlavourA>
    </xsl:template>
</xsl:stylesheet>
```

but we also need to create the following, very similar output:

```xml
<FlavourB>
    <book>
        <publisher>Wiley Group</publisher>
        <title>The Green Mile</title>
        <author>Stephen King</author>
        <price>18</price>
    </book>
    <book>
        <publisher>Wiley Group</publisher>
        <title>The Catcher in the Rye</title>
        <author>J. D. Salinger</author>
        <price>25</price>
    </book>
    <book>
        <publisher>Wiley Group</publisher>
        <title>I, Robot</title>
        <author>Issac Asimov</author>
        <price>15</price>
    </book>
    <book>
        <publisher>Time Publishing</publisher>
        <title>Foundation Novels</title>
        <author>Isaac Asimov</author>
        <price>30</price>
    </book>
    <book>
        <publisher>Time Publishing</publisher>
        <title>Pygmalion</title>
        <author>Oscar Wilde</author>
        <price>10</price>
    </book>
    <book>
        <publisher>Amazon</publisher>
        <title>Dune</title>
        <author>Frank Herbert</author>
        <price>25</price>
    </book>
    <book>
        <publisher>Amazon</publisher>
        <title>Dune Messiah</title>
        <author>Frank Herbert</author>
        <price>12</price>
    </book>
</FlavourB>
```

which we can generate with the following xslt:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <FlavourA>
            <xsl:for-each select="library/publisher">
                <xsl:variable name="publisher" select="." as="element(publisher)"/>
                <xsl:for-each select="book">
                    <book publisher="{$publisher/@name}">
                        <title>{title/text()}</title>
                        <author>{author/text()}</author>
                        <price>{price/text()}</price>
                    </book>
                </xsl:for-each>  
            </xsl:for-each>
        </FlavourA>
    </xsl:template>
</xsl:stylesheet>
```

Clearly, flavourA and flavourB are very similar, and we don't want to duplicate effort and time developing two almost identical pieces of software, so how do we proceed?

The only real difference here is how each transform constructs the book element (we could try to boil it down further to the specific 'publisher' element vs attribute, but lets keep it simple for the moment). This code though is embedded in both stylesheets, we could have an xsl:choose and pass a parameter in, but this only creates a large and bloated code base, so we refactor, we attempt to isolate the change:

FlavourA now becomes:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:kooks="http://kookerella.com"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <FlavourA>
            <xsl:for-each select="library/publisher">
                <xsl:variable name="publisher" select="." as="element(publisher)"/>
                <xsl:for-each select="book">
                    <xsl:sequence select="kooks:makeBook($publisher/@name,title/text(),author/text(),price/text())"/>
                </xsl:for-each>  
            </xsl:for-each>
        </FlavourA>
    </xsl:template>

    <xsl:function name="kooks:makeBook" as="element(book)">
        <xsl:param name="publisher" as="xs:string"/>
        <xsl:param name="title" as="xs:string"/>
        <xsl:param name="author" as="xs:string"/>
        <xsl:param name="price" as="xs:string"/>
        <book>
            <publisher>{$publisher}</publisher>
            <title>{$title}</title>
            <author>{$author}</author>
            <price>{$price}</price>
        </book>
    </xsl:function>
</xsl:stylesheet>
```

and flavourB becomes this:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:kooks="http://kookerella.com"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <FlavourA>
            <xsl:for-each select="library/publisher">
                <xsl:variable name="publisher" select="." as="element(publisher)"/>
                <xsl:for-each select="book">
                    <xsl:sequence select="kooks:makeBook($publisher/@name,title/text(),author/text(),price/text())"/>
                </xsl:for-each>  
            </xsl:for-each>
        </FlavourA>
    </xsl:template>

    <xsl:function name="kooks:makeBook" as="element(book)">
        <xsl:param name="publisher" as="xs:string"/>
        <xsl:param name="title" as="xs:string"/>
        <xsl:param name="author" as="xs:string"/>
        <xsl:param name="price" as="xs:string"/>
        <book publisher="{$publisher}">
            <title>{$title}</title>
            <author>{$author}</author>
            <price>{$price}</price>
        </book>
    </xsl:function>
</xsl:stylesheet>
```

Now we have isolated the difference inside a single function, there are two versions of this function, with the subtly different book construction, but apart from that the code is syntactically identical.

So now we 'inject' (actually technically this is what is known as 'higher order functions' in functional programming, as we're only going to pass a function, not an object....see later blogs for 'objects'). If we can pass the different functions into this code then the caller can define which function to use to make the book, and the rest of the code can be shared, here is our shared stylesheet:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:kooks="http://kookerella.com"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>

    <xsl:template match="/" mode="shared">
        <xsl:param name="makeBook" as="function(xs:string,xs:string,xs:string,xs:string) as element(book)"/>
        <FlavourA>
            <xsl:for-each select="library/publisher">
                <xsl:variable name="publisher" select="." as="element(publisher)"/>
                <xsl:for-each select="book">
                    <xsl:sequence select="$makeBook($publisher/@name,title/text(),author/text(),price/text())"/>
                </xsl:for-each>  
            </xsl:for-each>
        </FlavourA>
    </xsl:template>
</xsl:stylesheet>
```

note the parameter, 'makeBook' with a scary looking type

```xml
function(xs:string,xs:string,xs:string,xs:string) as element(book)
```

and we now use this function parameter to create the book:

```xml
<xsl:sequence select="$makeBook($publisher/@name,title/text(),author/text(),price/text())"/>
```

We can now create two separate stylesheets that include this shared code, define how to create a book, and pass that to the shared code:

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:kooks="http://kookerella.com"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:include href="shared.xsl"/>  
    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <xsl:apply-templates select="/" mode="shared">
            <xsl:with-param name="makeBook" select="kooks:makeBookA#4"/>
        </xsl:apply-templates>
    </xsl:template>

    <xsl:function name="kooks:makeBookA" as="element(book)">
        <xsl:param name="publisher" as="xs:string"/>
        <xsl:param name="title" as="xs:string"/>
        <xsl:param name="author" as="xs:string"/>
        <xsl:param name="price" as="xs:string"/>
        <book>
            <publisher>{$publisher}</publisher>
            <title>{$title}</title>
            <author>{$author}</author>
            <price>{$price}</price>
        </book>
    </xsl:function>
</xsl:stylesheet>
```

and

```xml
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:kooks="http://kookerella.com"
    exclude-result-prefixes="xs"
    expand-text="yes"
    version="3.0">
    <xsl:include href="shared.xsl"/>

    <xsl:output method="xml" indent="true" encoding="UTF-8"/>
    <xsl:template match="/">
        <xsl:apply-templates select="/" mode="shared">
            <xsl:with-param name="makeBook" select="kooks:makeBookB#4"/>
        </xsl:apply-templates>
    </xsl:template>

    <xsl:function name="kooks:makeBookB" as="element(book)">
        <xsl:param name="publisher" as="xs:string"/>
        <xsl:param name="title" as="xs:string"/>
        <xsl:param name="author" as="xs:string"/>
        <xsl:param name="price" as="xs:string"/>
        <book publisher="{$publisher}">
            <title>{$title}</title>
            <author>{$author}</author>
            <price>{$price}</price>
        </book>
    </xsl:function>
</xsl:stylesheet>
```

and we're done.

This was obviously a contrived and overly simplistic example. More often than not, the issue isn't two flavours of a complete transformation, but the some variation on a common theme within a single transformation, maybe constructing tables of information, where the code for constructing the table is identical in each case, but the data that appears, or its formatting is subtly different.

Next time we'll find out that actually there is an alternative, one which XSLT is quite well suited to, but may also make you rethink how you approach injection in other languages.
