# CSS notes

## grid

[css tricks](https://css-tricks.com/snippets/css/complete-guide-grid/)

6 elements in each row, each element has 200px height

```css
.wrapper{
    display: grid;
    grid-template-columns: repeat(6, 1fr); /* six rows */
    grid-auto-rows: 200px;
}

.item {
  justify-self: start | end | center | stretch;
}
```

## flexbox

elements in wrapper are center-aligned horizontally and vertically

Note: use `align-content: flex-end;` if reversely aligned (like bottom-top)

`align-items` describes how items align themselves in current row(column)

```css
.wrapper{
    display: flex;
    flex-direction: column;
    justify-content: center; /* main axis */
    align-content: center /* cross axis */
}
```