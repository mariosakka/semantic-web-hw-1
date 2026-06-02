Oral Questions and Answers
==========================

These are questions that came up during the project review, answered based on what this project actually does.

1. What is an XML attribute on `id`? Why is it an attribute and not an element?
--------------------------------------------------------------------------------

In our XML a user looks like `<user id="u1" skillLevel="Beginner" preferredCuisine="Italian">`. The `id`, `skillLevel`, and `preferredCuisine` are attributes, while `name` and `surname` are child elements. Attributes suit short structured values used in XPath predicates. Text content like a name fits better as an element. You read attributes in Java with `element.getAttribute("id")` and child text with `getElementsByTagName("name").item(0).getTextContent()`.

2. Why did you use the Mozilla user-agent in the scraper?
----------------------------------------------------------

BBC Good Food returns a stripped page or blocks the request when it detects a non-browser client. Setting the User-Agent to a real Chrome string makes the server treat the request as a normal browser visit. Without it the CSS selector finds nothing and the scrape returns zero results.

3. Why does the scraper have @Service on it?
---------------------------------------------

`ScraperService` is annotated with `@Service` so it is a Spring-managed bean. Spring creates one instance at startup and injects `RecipeService` into it through the constructor. `RecipeController` declares `ScraperService` as a constructor dependency and Spring supplies it automatically.

4. What library do you use for the scraper?
--------------------------------------------

Jsoup, declared in `build.gradle`. It handles the HTTP request and parses the response HTML into a document you can query with CSS selectors.

5. What happens if you can't scrape 20 recipes?
-------------------------------------------------

The scraper counts how many it added from the web. If the total is still below 20, `addFromBackup()` reads `backup-recipes.xml` from the classpath and adds as many recipes from it as needed to reach 20. If the HTTP request fails entirely, the fallback fills all 20 from the backup.

6. What are the constructors for?
----------------------------------

Constructor injection. Spring sees a bean with a single constructor and automatically resolves the parameters from the application context. No `@Autowired` is needed. Examples: `RecipeService(XmlStore, XmlService)`, `UserService(XmlStore, XmlService)`, `ScraperService(RecipeService)`, `RecipeController(RecipeService, UserService, ScraperService)`.

7. Do you validate XML?
------------------------

Yes, on load. `DocumentBuilderFactory` is configured with `setValidating(true)` and a custom error handler that throws on any DTD violation. If the file does not match the DTD the app fails to start with an `IllegalStateException`. Validation does not run again on write, but the controller validates all input before it reaches the service so only legal values enter the DOM.

8. Where is frontend input validation done?
--------------------------------------------

In the private `validate()` method inside `RecipeController` and `UserController`. It checks that fields are not blank and that submitted strings match an enum label using `CuisineType.contains()` and `DifficultyLevel.contains()`. On failure the controller re-renders the form with an error message. There are no `@Valid` or `@NotBlank` annotations.

9. Where are the recommendation queries?
-----------------------------------------

In `RecipeService`:
- `recommendBySkill` runs `"//recipe[@difficulty='" + skillLevel + "']"`
- `recommendBySkillAndCuisine` runs `"//recipe[@difficulty='" + skillLevel + "' and cuisineTypes/cuisineType/@type='" + cuisine + "']"`

`UserController.populateRecommendations()` calls both and puts the results in the model under `bySkill` and `bySkillAndCuisine`.

10. What does the `$` mean in XPath queries?
---------------------------------------------

Does not apply to this project. XPath variable resolvers were removed. Queries concatenate values directly into the expression string.

11. Did you process the scraped titles in any way?
---------------------------------------------------

Yes. The scraper skips empty titles, titles containing `"premium piece of content"`, and titles starting with `"App only"`. Each title has `.trim()` applied. No manual XML escaping is needed because Java's DOM API escapes special characters automatically when you call `setTextContent()`.

12. What happens if you omit the user-agent?
---------------------------------------------

Jsoup sends its default Java HTTP client identifier. BBC Good Food either blocks the request or returns a page with no recipe cards, causing the CSS selector to match nothing and the scrape to add zero recipes. The fallback would then fill all 20 from `backup-recipes.xml`.

13. Do you indent the XML output?
-----------------------------------

Yes. `xmlService.save()` creates a `Transformer` and sets `OutputKeys.INDENT` to `"yes"` and `indent-amount` to `4`. Every time data is written to disk the file comes out with 4-space indentation.

14. Why did you use NodeList?
------------------------------

It is the type returned by XPath evaluation with `XPathConstants.NODESET` and by `getElementsByTagName()`. The DOM API works with it natively via `getLength()` and `item(i)`. We iterate it to map each node into a `Recipe` or `User` object.

15. Do we save instantly to the XML file once we add a recipe or user, or does that happen only after scraping?
---------------------------------------------------------------------------------------------------------------

Instantly — every time. `RecipeService.add()` appends the new element to the in-memory DOM and immediately calls `xmlService.save()` on the same line. `UserService.addUser()` does the same. There is no batching or deferred write.

This means the scraper triggers a full disk write for every single recipe it adds. If it adds 15 recipes from the web and 5 from the backup, the file is written 20 times. The same is true when a user submits the add-recipe form — the file is written once per submission.

16. How do we find the ID for a new recipe or user?
----------------------------------------------------

Both `RecipeService` and `UserService` have a private `nextId()` method. It queries all existing records with `//recipe` or `//user`, reads the `id` attribute on each one, strips the leading letter prefix with `substring(1)`, parses the remainder as an integer, and tracks the maximum. The new ID is that maximum plus one, re-prefixed with `"r"` or `"u"`.

```java
private String nextId() throws XPathExpressionException {
    NodeList nodes = xmlService.queryNodeList(xmlStore.getRecipesDoc(), "//recipe");
    int max = 0;
    for (int i = 0; i < nodes.getLength(); i++) {
        String idVal = ((Element) nodes.item(i)).getAttribute("id");
        try {
            max = Math.max(max, Integer.parseInt(idVal.substring(1)));
        } catch (NumberFormatException | IndexOutOfBoundsException ignored) {}
    }
    return "r" + (max + 1);
}
```

If no records exist yet, `max` stays 0 and the first ID is `"r1"` or `"u1"`. The try-catch silently skips any malformed ID rather than crashing the add operation.

17. Do you use `@PostConstruct` and what is it for?
----------------------------------------------------

Yes, on `XmlStore.init()`. Spring calls a `@PostConstruct` method automatically after it constructs the bean and injects all its dependencies, but before the application starts accepting requests. This ordering guarantee is the whole point: `XmlStore` depends on `XmlService`, so it cannot load the XML files in its constructor because `XmlService` has not been injected yet. `@PostConstruct` runs after injection is complete, making it safe to call `xmlService.load()`. `@PostConstruct` methods cannot declare checked exceptions, so any that are thrown during parsing are caught and wrapped in `IllegalStateException` to fail the startup cleanly.

18. How do we make sure a recipe does not have two identical cuisine types?
---------------------------------------------------------------------------

It depends on where the recipe comes from.

For **scraped recipes**, `ScraperService` uses a do-while loop to guarantee two distinct random indices into the `CuisineType` array:

```java
int i1 = rng.nextInt(cuisines.length);
int i2;
do { i2 = rng.nextInt(cuisines.length); } while (i2 == i1);
```

`i2` is re-drawn until it is different from `i1`, so the two cuisine types are always distinct.

For **backup recipes**, the values are hardcoded in `backup-recipes.xml`. There is no runtime check — it is the responsibility of whoever wrote the file to ensure the two `<cuisineType>` entries are different.

For **manually added recipes** via the add-recipe form, the private `validate()` method in `RecipeController` checks this explicitly:

```java
if (ct1.equals(ct2)) return "The two cuisine types must be different.";
```

If the user selects the same cuisine type twice, the controller rejects the submission and re-renders the form with that error message.

The DTD does not enforce this. It requires exactly two `<cuisineType>` elements but says nothing about them being distinct — the check lives entirely in the application layer.

19. How do we read and write XML in all cases?
----------------------------------------------

**Reading the main data files** (`recipes.xml`, `users.xml`) goes through `XmlService.load()`:

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setValidating(true);
DocumentBuilder builder = factory.newDocumentBuilder();
builder.setErrorHandler(...); // throws on DTD violations
return builder.parse(filePath.toFile());
```

DTD validation is on. A custom error handler throws on any violation so the app fails to start with corrupt data.

**Reading the backup file** (`backup-recipes.xml`) happens directly inside `ScraperService.addFromBackup()`:

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setValidating(false);
DocumentBuilder builder = factory.newDocumentBuilder();
builder.setEntityResolver((pub, sys) -> new InputSource(new StringReader("")));
Document doc = builder.parse(in);
```

Validation is off because the backup file has no DOCTYPE declaration. The entity resolver is stubbed to suppress any warnings about a missing DTD. The file is loaded from the classpath as an `InputStream`.

**Writing** always goes through `XmlService.save()`:

```java
Transformer transformer = TransformerFactory.newInstance().newTransformer();
transformer.setOutputProperty(OutputKeys.INDENT, "yes");
transformer.setOutputProperty("{http://xml.apache.org/xslt}indent-amount", "4");
transformer.setOutputProperty(OutputKeys.DOCTYPE_SYSTEM, doc.getDoctype().getSystemId());
transformer.transform(new DOMSource(doc), new StreamResult(filePath.toFile()));
```

The entire in-memory DOM is serialised to the file with 4-space indentation. The `DOCTYPE_SYSTEM` property preserves the `<!DOCTYPE ...>` declaration so the file stays valid on the next load.

20. How would you force the two cuisine types to be different through the DTD?
-------------------------------------------------------------------------------

You cannot. DTD has no mechanism to compare the values of two sibling elements or attributes against each other. It can only constrain what elements exist and in what order, what attributes are present, and what each attribute's individual allowed values are.

To enforce that the two `<cuisineType type="..."/>` values are different you would need something more expressive — XML Schema (XSD) with an `xs:unique` constraint, or Schematron which lets you write arbitrary rules like `cuisineType[1]/@type != cuisineType[2]/@type`. DTD simply cannot express that kind of cross-element relationship. In this project the check lives in the application layer instead: the `validate()` method in `RecipeController` and the do-while loop in the scraper.

21. Do we compile XPath queries directly on the in-memory DOM?
--------------------------------------------------------------

No — compilation and evaluation are two separate steps that happen to be chained on one line:

```java
XPath xpath = xpathFactory.newXPath();
(NodeList) xpath.compile(expression).evaluate(doc, XPathConstants.NODESET);
```

`compile(expression)` parses the XPath string into an `XPathExpression` object. It checks the syntax and builds an internal representation of the query but does not touch the DOM at all.

`evaluate(doc, XPathConstants.NODESET)` is the step that actually traverses the in-memory DOM and returns results.

The two steps could be written separately and the behaviour would be identical:

```java
XPathExpression expr = xpath.compile(expression);
NodeList nodes = (NodeList) expr.evaluate(doc, XPathConstants.NODESET);
```

The `XPathFactory` is stored as a field in `XmlService` and reused across calls. A new `XPath` instance is created per call and the compiled `XPathExpression` is not cached — it is compiled fresh every time a query runs.

22. How are CSS styles defined? What are `@apply` and `@layer`?
---------------------------------------------------------------

Tailwind CSS is loaded via a CDN `<script>` tag in `fragments/head.html`. All component class definitions live in a `<style type="text/tailwindcss">` block in the same file, so they are processed by the Tailwind browser build rather than a build step.

`@layer components { }` places the rules inside Tailwind's "components" layer. Tailwind organises styles into three layers — `base`, `components`, and `utilities` — in that specificity order. Putting custom classes in `components` means Tailwind utility classes (which sit in `utilities`) can still override them when needed.

`@apply` composes existing Tailwind utility classes into a named CSS class:

```css
.btn { @apply px-4 py-2 bg-gray-800 text-white text-sm rounded hover:bg-gray-700; }
```

This defines `.btn` as a shorthand for that set of utilities. Templates then use only the short class name (`class="btn"`) rather than repeating the full utility string on every button. All class definitions — layout, sidebar, typography, form controls, tables, cards — are centralised in `head.html`. No template file contains raw Tailwind utility strings.

23. How do we apply XSLT?
--------------------------

`RecipeService.applyDisplayXslt()` loads the XSL file from the classpath and passes it to `XmlService.applyXslt()`:

```java
public String applyDisplayXslt(String skillLevel) throws TransformerException {
    InputStream xsl = getClass().getClassLoader().getResourceAsStream("xslt/recipes-display.xsl");
    return xmlService.applyXslt(xmlStore.getRecipesDoc(), xsl, Map.of("skill-level", skillLevel));
}
```

Inside `applyXslt()`:

```java
TransformerFactory tf = TransformerFactory.newInstance();
Transformer transformer = tf.newTransformer(new StreamSource(xslStream));
params.forEach(transformer::setParameter);
StringWriter writer = new StringWriter();
transformer.transform(new DOMSource(doc), new StreamResult(writer));
return writer.toString();
```

1. `TransformerFactory.newTransformer(new StreamSource(xslStream))` compiles the XSL file into a `Transformer`.
2. `setParameter` passes the `skill-level` value so the stylesheet can use `$skill-level` to colour rows.
3. `transform(new DOMSource(doc), new StreamResult(writer))` runs the transformation — takes the in-memory DOM as input and writes the output to a `StringWriter`.
4. The `StringWriter` is converted to a plain HTML string and returned.

The controller puts that string in the model under `xsltHtml` and the Thymeleaf template injects it with `th:utext`.

24. What is the class loader and how does it work?
---------------------------------------------------

The class loader is the JVM mechanism responsible for finding and loading `.class` files and resources into memory at runtime. Every class in a running Java application was loaded by a class loader.

When you call:

```java
getClass().getClassLoader().getResourceAsStream("xslt/recipes-display.xsl")
```

- `getClass()` returns the `Class` object for the current class
- `getClassLoader()` returns the class loader that loaded it
- `getResourceAsStream("xslt/recipes-display.xsl")` searches the classpath for a resource at that path and returns it as an `InputStream`

In a Spring Boot app, everything under `src/main/resources/` is on the classpath. So `"xslt/recipes-display.xsl"` resolves to `src/main/resources/xslt/recipes-display.xsl` and `"data/backup-recipes.xml"` resolves to `src/main/resources/data/backup-recipes.xml`.

The important difference from `new File("src/main/resources/xslt/recipes-display.xsl")` is that a hardcoded file path only works when running from the project root directory. The class loader approach works anywhere — including when the app is packaged as a JAR, because the class loader knows how to read resources from inside the JAR file itself.

25. What is a StringWriter?
----------------------------

`StringWriter` is a `java.io` class that implements the `Writer` interface but writes to an in-memory `StringBuffer` instead of a file or network socket. Anything written to it accumulates in memory and can be retrieved as a plain `String` via `toString()`.

In `XmlService.applyXslt()` it is used as the destination for the XSLT transformation:

```java
StringWriter writer = new StringWriter();
transformer.transform(new DOMSource(doc), new StreamResult(writer));
return writer.toString();
```

`StreamResult` accepts any `Writer` as its destination. By passing a `StringWriter` instead of a file, the entire HTML output of the transformation is captured in memory and returned as a string rather than written to disk.

26. What is an InputStream?
-----------------------------

`InputStream` is a `java.io` abstract class representing a source of bytes that can be read sequentially. It is the standard Java abstraction for any byte source — a file, a network connection, a resource inside a JAR, or a byte array in memory.

In this project it appears in two places:

- `getClassLoader().getResourceAsStream(...)` returns an `InputStream` pointing to a classpath resource. `applyXslt()` wraps it in a `StreamSource` so the XSLT transformer can read the XSL file byte by byte.
- `addFromBackup()` receives the backup XML file as an `InputStream` from the classpath and passes it directly to `builder.parse(in)`.

The key property of `InputStream` is that it abstracts away where the bytes come from. The `DocumentBuilder` and `TransformerFactory` do not need to know whether they are reading from a file on disk or from inside a JAR — they just read bytes from the stream.
