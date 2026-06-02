XML Runtime Guide
=================

DTD validation at runtime
--------------------------

When the app starts, `XmlStore` loads both data files by calling `xmlService.load()`. Inside that method, `DocumentBuilderFactory` is configured with `setValidating(true)` before a `DocumentBuilder` is created:

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setValidating(true);
DocumentBuilder builder = factory.newDocumentBuilder();
```

With validation enabled, the parser reads the DOCTYPE declaration at the top of each XML file and locates the referenced DTD. For `recipes.xml` that line is:

```xml
<!DOCTYPE recipes SYSTEM "recipes.dtd">
```

The parser resolves `recipes.dtd` relative to the XML file's location. It then checks every element and attribute against the rules defined in the DTD — rejecting a file if a recipe is missing `difficulty`, if a `difficulty` value is not in the enumeration, or if a `cuisineType/@type` is not one of the ten allowed tokens.

A custom error handler captures both warnings and errors:

```java
builder.setErrorHandler(new org.xml.sax.helpers.DefaultHandler() {
    @Override
    public void warning(org.xml.sax.SAXParseException e) {
        log.warn("XML validation warning: {}", e.getMessage());
    }

    @Override
    public void error(org.xml.sax.SAXParseException e) throws SAXException {
        throw e;
    }
});
```

`warning()` logs non-fatal issues. `error()` re-throws the exception so DTD violations abort the load. `XmlStore` catches the resulting `SAXException` and wraps it in an `IllegalStateException`, stopping the application from starting with corrupt data.

Validation runs only on load. When the app writes new data via `xmlService.save()`, it does not re-validate. The controller layer validates all input before calling the service, so only values already in the DTD enumeration reach the XML.

Annotations
-----------

`@SpringBootApplication` on `BigHw1Application` — shorthand for configuration scanning, auto-configuration, and component scanning. One annotation is enough to start a fully working Spring application.

`@Component` on `XmlStore` — registers it as a Spring-managed bean. Spring creates one instance at startup and holds it for the lifetime of the process.

`@Service` on `XmlService`, `RecipeService`, `UserService`, `ScraperService` — identical to `@Component`. The distinction is semantic: `@Service` signals business or data-access logic.

`@Controller` on `RecipeController` and `UserController` — registers the class and routes HTTP requests to its methods. Methods returning a string render the matching Thymeleaf template; strings starting with `"redirect:"` send a 302.

`@PostConstruct` on `XmlStore.init()` — Spring calls this method after constructing the bean and injecting dependencies, before the application accepts requests. This is where the XML files are loaded into memory. Checked exceptions cannot be declared, so they are caught and wrapped in `IllegalStateException`:

```java
@PostConstruct
public void init() {
    recipesPath = Paths.get("src/main/resources/data/recipes.xml");
    usersPath   = Paths.get("src/main/resources/data/users.xml");
    try {
        recipesDoc = xmlService.load(recipesPath);
        usersDoc   = xmlService.load(usersPath);
    } catch (ParserConfigurationException | SAXException | IOException e) {
        throw new IllegalStateException("Failed to load XML data files", e);
    }
}
```

`@GetMapping` / `@PostMapping` — bind a method to an HTTP verb and URL path.

`@RequestParam` — extracts a named value from the query string or form body.

`@PathVariable` — extracts a segment from the URL path. A request to `/recipes/r3` with `@GetMapping("/recipes/{id}")` sets the `id` parameter to `"r3"`.

XML output formatting
---------------------

Every time data is written to disk, `xmlService.save()` produces indented output. The `Transformer` is configured before the transform runs:

```java
transformer.setOutputProperty(OutputKeys.INDENT, "yes");
transformer.setOutputProperty("{http://xml.apache.org/xslt}indent-amount", "4");
transformer.setOutputProperty(OutputKeys.DOCTYPE_SYSTEM, doc.getDoctype().getSystemId());
```

`OutputKeys.INDENT` set to `"yes"` enables indentation. The vendor-specific `indent-amount` property controls how many spaces per level — 4 in this case. `DOCTYPE_SYSTEM` preserves the `<!DOCTYPE ...>` declaration so the file stays valid and the parser can find the DTD on the next load.

Frontend input validation
--------------------------

Controller-level validation happens in a private `validate()` method inside `RecipeController` and `UserController`. It runs before any service call. What it checks:

- Fields are not blank.
- Submitted cuisine strings match an enum label: `CuisineType.contains(ct1)`.
- Submitted difficulty strings match an enum label: `DifficultyLevel.contains(difficulty)`.

```java
private String validate(String title, String ct1, String ct2, String difficulty) {
    if (title.isBlank()) return "Title is required.";
    if (!CuisineType.contains(ct1)) return "Invalid cuisine type.";
    if (!DifficultyLevel.contains(difficulty)) return "Invalid difficulty.";
    return "";
}
```

On failure the controller re-renders the add form and passes an error message in the model. The submitted field values are also passed back so the user does not have to retype them. On success the controller calls the service and redirects.

There are no `@Valid` or `@NotBlank` annotations in this project. Spring's Bean Validation integration is not used.

XSLT transformation
--------------------

`XmlService.applyXslt()` takes the in-memory DOM document, an InputStream pointing to the XSL file, and a map of parameter values:

```java
public String applyXslt(Document doc, InputStream xslStream, Map<String, String> params)
        throws TransformerException {
    TransformerFactory tf = TransformerFactory.newInstance();
    Transformer transformer = tf.newTransformer(new StreamSource(xslStream));
    if (params != null) {
        params.forEach(transformer::setParameter);
    }
    StringWriter writer = new StringWriter();
    transformer.transform(new DOMSource(doc), new StreamResult(writer));
    return writer.toString();
}
```

`TransformerFactory.newTransformer()` compiles the stylesheet. Parameters are applied with `setParameter`, which is how the `skill-level` value reaches the XSL template. The result is written to a `StringWriter` and returned as a plain HTML string.

`RecipeService.applyDisplayXslt()` loads the XSL file from the classpath and calls this method:

```java
public String applyDisplayXslt(String skillLevel) throws TransformerException {
    InputStream xsl = getClass().getClassLoader().getResourceAsStream("xslt/recipes-display.xsl");
    return xmlService.applyXslt(xmlStore.getRecipesDoc(), xsl, Map.of("skill-level", skillLevel));
}
```

The controller puts the resulting HTML string into the model under `xsltHtml`. The Thymeleaf template injects it with `th:utext`, which inserts the raw HTML without escaping:

```html
<div th:utext="${xsltHtml}"></div>
```
