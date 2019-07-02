# Config

## In every HTML

```html
<html xmlns:th="http://www.thymeleaf.org">
```

## Spring Integration

[thymeleaf-spring github](https://github.com/thymeleaf/thymeleaf-spring)

Note: do NOT use ServletContextTemplateResolver in offcial doc, it's in thymeleaf package.
Use view resolver in thymeleaf-spring module.

If you are not using spring 4, check github for config.

```xml
<bean id="templateResolver"
    class="org.thymeleaf.spring4.templateresolver">
<property name="prefix" value="/WEB-INF/templates/" />
<property name="suffix" value=".html" />
<property name="templateMode" value="HTML5" />
</bean>

<bean id="templateEngine"
    class="org.thymeleaf.spring4.SpringTemplateEngine">
<property name="templateResolver" ref="templateResolver" />
</bean>

<bean class="org.thymeleaf.spring4.view.ThymeleafViewResolver">
<property name="templateEngine" ref="templateEngine" />
</bean>
```