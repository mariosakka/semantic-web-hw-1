XPath Query Guide
=================

All XPath expressions are evaluated against the in-memory DOM held by `XmlStore` using the `javax.xml.xpath` API. Values are concatenated directly into the expression string.

`difficulty` and cuisine type values are stored as XML attributes. All expressions therefore use `@difficulty` and `cuisineTypes/cuisineType/@type` to access these fields.

Core syntax
-----------

`//` selects matching nodes anywhere in the document regardless of depth. Used throughout because it avoids hard-coding the path from the root.

`[@attr='value']` is a predicate that filters to elements whose attribute equals the given value. `@` means attribute rather than child element.

`cuisineTypes/cuisineType/@type` navigates from the `<recipe>` context node down to its `<cuisineType>` grandchildren and reads their `type` attribute. The predicate is satisfied if any of the two `<cuisineType>` elements matches.

`and` inside a predicate joins two conditions — both must be true.

`[1]` selects the first node in the result set. XPath positions are 1-based.

Queries
-------

**Get all recipes** — `RecipeService.getAll()`
```java
"//recipe"
```

**Get recipe by id** — `RecipeService.getById(String id)`
```java
"//recipe[@id='" + id + "']"
```

**Filter by cuisine** — `RecipeService.filterByCuisine(String cuisine)`
```java
"//recipe[cuisineTypes/cuisineType/@type='" + cuisine + "']"
```
Matches if either of the two `<cuisineType>` children has a `type` equal to the value.

**Recommend by skill** — `RecipeService.recommendBySkill(String skillLevel)`
```java
"//recipe[@difficulty='" + skillLevel + "']"
```

**Recommend by skill and cuisine** — `RecipeService.recommendBySkillAndCuisine(String skillLevel, String cuisine)`
```java
"//recipe[@difficulty='" + skillLevel + "' and cuisineTypes/cuisineType/@type='" + cuisine + "']"
```
The primary recommendation query. Returns recipes that match both conditions.

**Get all users** — `UserService.getAll()`
```java
"//user"
```

**Get first user** — `UserService.getFirstUser()`
```java
"//user[1]"
```
Used as the default selection on the recommendations page.

**Get user by id** — `UserService.getById(String id)`
```java
"//user[@id='" + id + "']"
```

The `$` syntax in XPath
-----------------------

In standard XPath, `$varname` is a variable reference — it tells the XPath engine to substitute a bound value rather than treating the name as a literal string. Java's `XPath` API supports this via a `XPathVariableResolver`. This project does not use that pattern. All values are concatenated directly into the expression string:

```java
"//recipe[@id='" + id + "']"
```

This is simpler but means a value containing a single quote could break the expression. For this project the values come from a DTD-validated enum set so no special characters can appear.

How queries run in Java
-----------------------

```java
XPath xpath = xpathFactory.newXPath();
NodeList nodes = (NodeList) xpath.compile(expression).evaluate(doc, XPathConstants.NODESET);
```

`XPathConstants.NODESET` tells the evaluator to return a `NodeList`. For single-node lookups the constant is `XPathConstants.NODE`, which returns a `Node` or null. The `XPathFactory` instance is reused — stored as a field in `XmlService`.

NodeList
--------

`NodeList` is the type returned by XPath evaluation with `XPathConstants.NODESET` and by `getElementsByTagName()`. It is the DOM API's native list type. There is no way to get a `List<Node>` directly from these calls.

Iterating it:

```java
for (int i = 0; i < nodes.getLength(); i++) {
    Node node = nodes.item(i);
    // cast to Element to access attributes and child elements
    Element el = (Element) node;
}
```

`getLength()` returns the count. `item(i)` returns the node at that index. In the services, each node is cast to `Element` and then passed to `nodeToRecipe()` or `nodeToUser()`, which read the attributes and child text to build a model object.
