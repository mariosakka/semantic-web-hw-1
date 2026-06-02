Thymeleaf Template Guide
========================

Thymeleaf processes templates on the server, replaces `th:*` attributes with computed values, and returns plain HTML. Templates are valid HTML files on their own.

Core attributes
---------------

`th:text="${expr}"` replaces the element's text content with the evaluated expression and HTML-escapes the result.

`th:utext="${expr}"` sets the element's inner HTML without escaping. Used only for `xsltHtml`, where the XSLT output is trusted HTML that must not be escaped. Never use `th:utext` with user input.

`th:each="item : ${list}"` repeats the element once per item. Inside the element, `${item.property}` accesses properties via the getter.

`th:if="${expr}"` removes the element from output when the expression is false. `th:unless` is the inverse.

`th:value="${expr}"` sets the `value` attribute of an input. Used to restore submitted values when a form fails validation.

`th:selected="${expr}"` adds the `selected` attribute when the expression is true.

`@{/path/{var}(var=${expr})}` is a URL expression that builds a context-aware link. `@{/recipes/{id}(id=${recipe.id})}` produces `/recipes/r3` when `recipe.id` is `"r3"`.

`th:replace="~{fragments/sidebar :: nav}"` replaces the host element with a named fragment from another template. The path is relative to `src/main/resources/templates/`.

Fragment inclusion
------------------

Every page includes two shared fragments:

```html
<head th:replace="~{fragments/head :: head('Page Title')}"></head>
<aside th:replace="~{fragments/sidebar :: nav}"></aside>
```

The head fragment contains the Tailwind CDN script and a `<style type="text/tailwindcss">` block with `@apply` component classes (`.btn`, `.card`, `.tbl`, `.nav-link`). All styling is defined once there; templates use only the short class names.

Common patterns
---------------

**Error message** (shown only on failed submission):
```html
<p class="error" th:if="${errorMsg != ''}" th:text="${errorMsg}"></p>
```

**Recipe table row:**
```html
<tr th:each="recipe : ${recipes}">
    <td><a th:href="@{/recipes/{id}(id=${recipe.id})}" th:text="${recipe.id}"></a></td>
    <td th:text="${recipe.title}"></td>
</tr>
```

**Cuisine dropdown with pre-selection:**
```html
<option th:each="ct : ${cuisineTypes}"
        th:value="${ct.label}"
        th:text="${ct.label}"
        th:selected="${ct.label == cuisineType1}">
</option>
```
`th:selected` compares `ct.label` (a String) to the submitted form value, not `ct` (an enum). Comparing an enum object to a String with `==` in SpEL always returns false because they are different types.

**XSLT output:**
```html
<div th:utext="${xsltHtml}"></div>
```

**User dropdown with compound label:**
```html
<option th:each="u : ${users}"
        th:value="${u.id}"
        th:text="${u.name} + ' ' + ${u.surname} + ' (' + ${u.skillLevel} + ')'"
        th:selected="${u.id == selectedUserId}">
</option>
```

**Null guard:**
```html
<div th:if="${user == null}">
    <p>No users found. <a href="/users/add">Add a user</a> first.</p>
</div>
```
Thymeleaf evaluates `== null` safely without throwing a NullPointerException.

**Conditional table:**
```html
<table th:if="${!recipes.isEmpty()}">...</table>
<p th:if="${recipes.isEmpty()}">No recipes found.</p>
```
