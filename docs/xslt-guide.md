XSLT Stylesheet Guide
=====================

`recipes-display.xsl` transforms `recipes.xml` into an HTML table. `XmlService.applyXslt()` applies it when the display page is submitted, passing the selected user's skill level as a parameter so the stylesheet can colour rows accordingly.

`difficulty` and cuisine types are stored as XML attributes, so the stylesheet uses `@difficulty` and `cuisineTypes/cuisineType[N]/@type` throughout.

Key XSLT elements
-----------------

**Parameter declaration**
```xml
<xsl:param name="skill-level" select="''"/>
```
Declares a top-level parameter that callers set before the transformation. `select="''"` gives it an empty string default. `XmlService.applyXslt()` sets the value with `transformer.setParameter("skill-level", skillLevel)`.

**Root template**
```xml
<xsl:template match="/">
    <table ...>
        <thead>...</thead>
        <tbody>
            <xsl:apply-templates select="//recipe"/>
        </tbody>
    </table>
</xsl:template>
```
`match="/"` is the entry point. It builds the static table skeleton and delegates each `<recipe>` node to the recipe template.

**Conditional variable**
```xml
<xsl:variable name="rowColor">
    <xsl:choose>
        <xsl:when test="@difficulty = $skill-level">#FFFF99</xsl:when>
        <xsl:otherwise>#90EE90</xsl:otherwise>
    </xsl:choose>
</xsl:variable>
```
`@difficulty` reads the attribute on the current context node. If it equals `$skill-level` the row is yellow; otherwise green.

**Attribute value template**
```xml
<tr style="background-color:{$rowColor};">
```
Curly braces inside an attribute value are evaluated at runtime and replaced with the variable's string value.

**Value extraction**
```xml
<xsl:value-of select="@id"/>
<xsl:value-of select="@difficulty"/>
<xsl:value-of select="cuisineTypes/cuisineType[1]/@type"/>
<xsl:value-of select="cuisineTypes/cuisineType[2]/@type"/>
```
`[1]` and `[2]` select the first and second `<cuisineType>` children by position.

Full recipe template
--------------------

```xml
<xsl:template match="recipe">
    <xsl:variable name="rowColor">
        <xsl:choose>
            <xsl:when test="@difficulty = $skill-level">#FFFF99</xsl:when>
            <xsl:otherwise>#90EE90</xsl:otherwise>
        </xsl:choose>
    </xsl:variable>
    <tr style="background-color:{$rowColor};">
        <td><xsl:value-of select="@id"/></td>
        <td><xsl:value-of select="title"/></td>
        <td><xsl:value-of select="cuisineTypes/cuisineType[1]/@type"/></td>
        <td><xsl:value-of select="cuisineTypes/cuisineType[2]/@type"/></td>
        <td><xsl:value-of select="@difficulty"/></td>
    </tr>
</xsl:template>
```

The result is an HTML string returned to `RecipeController`, stored in the model under `xsltHtml`, and injected into the page with `th:utext` (which does not escape the markup).
