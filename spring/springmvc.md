# Spring MVC

## Image response

```html
<img src = "getImage/id">
```

```java
@RequestMapping("/getImage/#{id}")
public void getImage(HttpServletResponse resp, @PathVariable String id){
    response.setContentType(MediaType.IMAGE_JPEG_VALUE);
    byte[] bimg = service.getPojoById(id).getImage();
    org.apache.commons.io.IOUtils.copy(new ByteArrayInputStream(bimg), resp.getOutputStream());
}
```

Note: if you save image as BLOB in MySQL, you can use byte[] to do ORM directly in Mybatis.

## Send data to View

```java
@RequestMapping("/dog")
public ModelAndView dogPage(){
    ModelAndView mv = new ModelAndView("dog");
    mv.addObject("dogs", dogList);
    return mv;
}
```

```java
@RequestMapping("/dog")
public String dogPage(){
    Model model = new Model("dog");
    model.addObject("dogs", dogList);
    return "dog";
}
```

