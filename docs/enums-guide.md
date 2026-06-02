Java Enums Guide
================

`CuisineType` and `DifficultyLevel` in `org.bighw1.big_hw_1.enums` replace the duplicated `List.of(...)` string constants that would otherwise appear across controllers and services.

Structure
---------

```java
public enum CuisineType {
    ITALIAN("Italian"),
    MIDDLE_EASTERN("Middle-Eastern");

    private final String label;

    CuisineType(String label) { this.label = label; }
    public String getLabel() { return label; }

    @Override
    public String toString() { return label; }

    public static boolean contains(String value) {
        for (CuisineType ct : values()) {
            if (ct.label.equals(value)) return true;
        }
        return false;
    }
}
```

Each constant carries a `label` string — the value stored in XML and submitted by HTML forms. `toString()` returns the label so that Thymeleaf's `th:text="${ct}"` and `th:value="${ct}"` produce the label string automatically. The label for `MIDDLE_EASTERN` is `"Middle-Eastern"` (hyphen, not space) because DTD enum tokens must be valid XML Nmtokens.

`contains(String value)` validates form input. Controllers call `CuisineType.contains(ct1)` to check whether a submitted string is an allowed value.

`DifficultyLevel` follows the same pattern with three constants: `BEGINNER("Beginner")`, `INTERMEDIATE("Intermediate")`, `ADVANCED("Advanced")`.

Constants
---------

`CuisineType` — 10 constants: `ITALIAN`, `ASIAN`, `MEXICAN`, `FRENCH`, `MEDITERRANEAN`, `INDIAN`, `AMERICAN`, `BRITISH`, `MIDDLE_EASTERN`, `GREEK`. Labels map one-to-one to the values allowed by `cuisineType/@type` in `recipes.dtd` and `preferredCuisine` in `users.dtd`.

`DifficultyLevel` — 3 constants. Labels map to `difficulty` in `recipes.dtd` and `skillLevel` in `users.dtd`.

Usage in templates
------------------

```java
model.addAttribute("cuisineTypes", CuisineType.values());
model.addAttribute("difficulties", DifficultyLevel.values());
```

```html
<option th:each="ct : ${cuisineTypes}"
        th:value="${ct.label}"
        th:text="${ct.label}"
        th:selected="${ct.label == cuisineType1}">
</option>
```

`th:selected` compares `ct.label` (a String) to the form-submitted string in the model. Comparing the enum object directly (`ct == cuisineType1`) would always be false.

Usage in ScraperService
-----------------------

```java
CuisineType[] cuisines = CuisineType.values();
DifficultyLevel[] difficulties = DifficultyLevel.values();
ThreadLocalRandom rng = ThreadLocalRandom.current();

int i1 = rng.nextInt(cuisines.length);
int i2;
do { i2 = rng.nextInt(cuisines.length); } while (i2 == i1);

recipeService.add(
    title,
    cuisines[i1].toString(),
    cuisines[i2].toString(),
    difficulties[rng.nextInt(difficulties.length)].toString());
```

Two distinct cuisine indices are selected by rejecting and re-drawing when `i2` collides with `i1`. `ThreadLocalRandom` reuses a thread-local instance instead of constructing a new `Random` per call.
