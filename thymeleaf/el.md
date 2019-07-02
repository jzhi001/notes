# Common expressions

[learn from demos](http://itutorial.thymeleaf.org/)

```html
<label th:each="id,status : ${idList}" th:for="|id${status.index}|" th:text="${id.id}"></label>
```

```html
<img th:src = "@{/getPetImage/{id}(id=${pet.id})}">

<img src = "/getPetImage/123">
```