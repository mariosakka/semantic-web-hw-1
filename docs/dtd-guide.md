DTD Schema Guide
================

`recipes.dtd` and `users.dtd` define the structure of the two XML data files. The app uses `DocumentBuilderFactory.setValidating(true)` so both files are checked against their DTDs every time they are loaded.

Attributes vs elements
----------------------

In this schema, short structured values used in XPath predicates or DTD enumeration are attributes. Longer human-readable text content is a child element.

| Field | Form | Reason |
|---|---|---|
| `id`, `difficulty` | attribute | used in `[@id=...]` predicates; must be unique/enumerated |
| `skillLevel`, `preferredCuisine` | attribute | matched against recipe attributes in XPath |
| `cuisineType/@type` | attribute | enumerated, used in `[@type=...]` predicates |
| `title`, `name`, `surname` | child element | free text, no structural constraint needed |

In Java, attributes and child text are read differently:

```java
// attribute
String id = element.getAttribute("id");

// child element text
String name = el.getElementsByTagName("name").item(0).getTextContent().trim();
```

Root element
```xml
<!DOCTYPE recipes SYSTEM "recipes.dtd">
```
`SYSTEM` references the DTD by a relative path. The parser resolves it from the same directory as the XML file.

Element sequence
```xml
<!ELEMENT recipe (title, cuisineTypes)>
```
`(title, cuisineTypes)` is a sequence: both children must appear in that order.

Text content
```xml
<!ELEMENT title (#PCDATA)>
```
`#PCDATA` means the element contains only text. Nesting a child element inside `<title>` fails validation.

ID attribute
```xml
<!ATTLIST recipe id ID #REQUIRED>
```
`ID` enforces uniqueness across the whole document. `#REQUIRED` means the attribute must be present on every element.

Attribute enumeration
```xml
<!ATTLIST recipe
    id ID #REQUIRED
    difficulty (Beginner|Intermediate|Advanced) #REQUIRED>
```
The parser rejects any `difficulty` value not in the list. DTD enum tokens must be valid XML Nmtokens, which cannot contain spaces — so `Middle Eastern` is written as `Middle-Eastern` throughout the schema.

Empty element
```xml
<!ELEMENT cuisineType EMPTY>
<!ATTLIST cuisineType
    type (Italian|Asian|Mexican|French|Mediterranean|Indian|American|British|Middle-Eastern|Greek) #REQUIRED>
```
`EMPTY` means no content. The cuisine value is carried entirely by the `type` attribute. Self-closing syntax `<cuisineType type="Italian"/>` is the natural form.

Fixed cardinality
```xml
<!ELEMENT cuisineTypes (cuisineType, cuisineType)>
```
Listing `cuisineType` twice forces exactly two cuisine entries per recipe.

Zero-or-more root
```xml
<!ELEMENT recipes (recipe*)>
```
`*` means zero or more, so the file is valid when empty — which is the initial state before the scraper runs.

---

recipes.dtd
-----------

```xml
<!ELEMENT recipes (recipe*)>
<!ELEMENT recipe (title, cuisineTypes)>
<!ATTLIST recipe
    id ID #REQUIRED
    difficulty (Beginner|Intermediate|Advanced) #REQUIRED>
<!ELEMENT title (#PCDATA)>
<!ELEMENT cuisineTypes (cuisineType, cuisineType)>
<!ELEMENT cuisineType EMPTY>
<!ATTLIST cuisineType
    type (Italian|Asian|Mexican|French|Mediterranean|Indian|American|British|Middle-Eastern|Greek) #REQUIRED>
```

Valid recipe:
```xml
<recipe id="r1" difficulty="Beginner">
    <title>Mushroom stroganoff</title>
    <cuisineTypes>
        <cuisineType type="Asian"/>
        <cuisineType type="Middle-Eastern"/>
    </cuisineTypes>
</recipe>
```

users.dtd
---------

```xml
<!ELEMENT users (user*)>
<!ELEMENT user (name, surname)>
<!ATTLIST user
    id ID #REQUIRED
    skillLevel (Beginner|Intermediate|Advanced) #REQUIRED
    preferredCuisine (Italian|Asian|Mexican|French|Mediterranean|Indian|American|British|Middle-Eastern|Greek) #REQUIRED>
<!ELEMENT name (#PCDATA)>
<!ELEMENT surname (#PCDATA)>
```

`skillLevel` is matched against recipe `@difficulty` in XPath recommendation queries. `preferredCuisine` is matched against `cuisineTypes/cuisineType/@type` in the combined recommendation query.

Valid user:
```xml
<user id="u1" skillLevel="Intermediate" preferredCuisine="Italian">
    <name>Alice</name>
    <surname>Demo</surname>
</user>
```
