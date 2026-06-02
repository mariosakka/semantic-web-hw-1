Scraper Guide
=============

`ScraperService` pulls recipe titles from BBC Good Food using Jsoup and stores them via `RecipeService`. If the scrape returns fewer than 20 recipes, the remainder are read from a hardcoded backup file.

How the scrape works
--------------------

`scrapeFromWeb` fetches the BBC Good Food budget autumn collection page. The request sets a browser User-Agent header because the site returns a stripped-down page to requests that look like bots:

```java
Document page = Jsoup.connect(SCRAPE_URL)
        .userAgent("Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...")
        .timeout(15000)
        .get();
```

Recipe titles are selected with a CSS selector:

```java
Elements cards = page.select("h2.heading-4:not(.promotion-cards__title--heading)");
if (cards.isEmpty()) cards = page.select("h2");
```

On BBC Good Food, recipe titles appear in `<h2>` elements with the class `heading-4`. The `:not(...)` part excludes promotional headings that use the same class. If that selector finds nothing, the fallback selects all `<h2>` elements.

For each heading, the text is trimmed, then blank titles and known junk strings are skipped:

```java
String title = card.text().trim();
if (title.isEmpty()) continue;
if (title.contains("premium piece of content") || title.startsWith("App only")) continue;
```

No manual XML escaping is needed when saving. The DOM API's `setTextContent()` escapes special characters (like `&` and `<`) automatically, so a recipe title containing those characters is stored correctly in the XML.

Titles that pass are saved via `recipeService.add()`. Since BBC Good Food does not publish cuisine types or difficulty levels in a structured way, both cuisine values are picked at random from `CuisineType` and the difficulty from `DifficultyLevel`. The two cuisine types are always different:

```java
int i1 = rng.nextInt(cuisines.length);
int i2;
do { i2 = rng.nextInt(cuisines.length); } while (i2 == i1);
```

`scrapeFromWeb` returns the number of recipes it added. Exceptions are caught, a warning is logged, and the method returns however many it managed to add before the failure.

The backup fallback
-------------------

After scraping, `scrapeIfNeeded` checks the total. If still below 20, it calls `addFromBackup` with the number still needed:

```java
int total = existing.size() + scraped;
if (total < 20) {
    addFromBackup(20 - total);
}
```

`addFromBackup` loads `backup-recipes.xml` from the classpath. This file contains 20 hardcoded recipes with real titles, cuisine types, and difficulty levels. It is parsed with a non-validating `DocumentBuilder` because it has no DOCTYPE declaration. The entity resolver is stubbed out to suppress warnings:

```java
factory.setValidating(false);
builder.setEntityResolver((pub, sys) -> new InputSource(new StringReader("")));
```

The method reads recipe elements in file order and calls `recipeService.add()` for as many as are needed:

```java
String title = el.getElementsByTagName("title").item(0).getTextContent().trim();
NodeList ctNodes = el.getElementsByTagName("cuisineType");
String ct1 = ((org.w3c.dom.Element) ctNodes.item(0)).getAttribute("type");
String ct2 = ((org.w3c.dom.Element) ctNodes.item(1)).getAttribute("type");
String difficulty = el.getAttribute("difficulty");
recipeService.add(title, ct1, ct2, difficulty);
```

If the backup file is missing or parsing fails, a warning is logged and the method returns without crashing.

When scraping is triggered
--------------------------

`scrapeIfNeeded` is called by `RecipeController.seed()`, which handles POST requests to `/admin/seed`. It checks whether the store already has 20 or more recipes before doing anything. If it does, it returns immediately.
