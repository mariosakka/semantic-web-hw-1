Spring Boot Application Guide
==============================

This note covers the application structure, annotation meanings, startup sequence, and request cycle.

Annotations
-----------

`@SpringBootApplication` on `BigHw1Application` is shorthand for three things: marks the class as a configuration source, enables auto-configuration (sets up Tomcat, Thymeleaf, and other defaults by inspecting the classpath), and triggers component scanning of the current package and all sub-packages.

`@Component` on `XmlStore` registers it as a Spring-managed bean. Spring creates one instance at startup and keeps it for the lifetime of the process.

`@Service` on `XmlService`, `RecipeService`, `UserService`, and `ScraperService` works identically to `@Component`. The difference is semantic: `@Service` signals business or data-access logic.

`@Controller` on `RecipeController` and `UserController` registers the class and tells Spring's dispatcher to route HTTP requests to its methods. A method returning a string like `"recipes/list"` causes Spring to render `src/main/resources/templates/recipes/list.html`. A string starting with `"redirect:"` sends a 302 to the browser.

`@PostConstruct` on `XmlStore.init()` marks a method Spring calls after constructing the bean and injecting its dependencies, before the application accepts requests. XML files are loaded here.

`@GetMapping` and `@PostMapping` bind a method to an HTTP verb and URL path.

`@RequestParam` extracts a named value from the query string or form body:
```java
@PostMapping("/recipes/add")
public String addSubmit(@RequestParam("title") String title, ...) { ... }
```

`@PathVariable` extracts a segment from the URL path:
```java
@GetMapping("/recipes/{id}")
public String recipeDetail(@PathVariable("id") String id, Model model) { ... }
```
A request to `/recipes/r3` sets `id` to `"r3"`.

Constructor injection
---------------------

```java
@Service
public class RecipeService {
    private final XmlStore xmlStore;
    private final XmlService xmlService;

    public RecipeService(XmlStore xmlStore, XmlService xmlService) {
        this.xmlStore = xmlStore;
        this.xmlService = xmlService;
    }
}
```
When Spring sees a bean with a single constructor it resolves the parameters from the application context automatically. No `@Autowired` is needed. Fields are `final` because they are set once and never change.

Startup sequence
----------------

```
SpringApplication.run() starts the context
    → XmlService is instantiated (no dependencies)
    → XmlStore is instantiated, @PostConstruct loads recipes.xml and users.xml into memory
    → RecipeService, UserService are instantiated
    → ScraperService is instantiated (depends on RecipeService)
    → RecipeController, UserController are instantiated
    → Tomcat begins accepting requests
```

Seeding is not automatic. The **Scrape or Seed** button on the home page POSTs to `/admin/seed`, which calls `scraper.scrapeIfNeeded()` and `userService.seedDefaultUsers()`.

Request cycle
-------------

A GET to `/recipes`:

1. Tomcat passes the request to Spring's `DispatcherServlet`.
2. `DispatcherServlet` finds `RecipeController.listRecipes()` via `@GetMapping("/recipes")`.
3. `recipeService.getAll()` runs an XPath query against the in-memory DOM and returns a `List<Recipe>`.
4. The list is added to the model under `"recipes"` and the method returns `"recipes/list"`.
5. Thymeleaf renders `src/main/resources/templates/recipes/list.html` with the model.
6. The rendered HTML is returned as the response body.

A POST to `/recipes/add` follows the same path but returns `"redirect:/recipes"` on success, which triggers a new GET from the browser.

Enums
-----

`CuisineType` and `DifficultyLevel` are plain Java enums, not Spring beans. They are loaded by the JVM at class-init time. Controllers pass them to templates via:
```java
model.addAttribute("cuisineTypes", CuisineType.values());
model.addAttribute("difficulties", DifficultyLevel.values());
```
Thymeleaf iterates them with `th:each` the same way it iterates a list.

Project dependencies (build.gradle)
-------------------------------------

```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
implementation 'org.jsoup:jsoup:1.17.2'
```

`spring-boot-starter-web` pulls in Spring MVC and embedded Tomcat. `spring-boot-starter-thymeleaf` adds the template engine and configures it to look in `src/main/resources/templates/`. Jsoup is declared separately because it is not part of any starter.
