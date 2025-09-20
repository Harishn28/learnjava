# Evolution of Java Web Development

## 1. CGI (Common Gateway Interface)
- Early way of handling web requests.
- Each request starts a new process ➞ very slow.
- Example: Perl/Python/Java program invoked per request.
- **Dependencies**: None, OS-level executables.

---

## 2. JSP (JavaServer Pages)
- Embeds Java code inside HTML.
- Faster than CGI, but mixes business logic with presentation ➞ messy.
- **Dependencies**: `javax.servlet.jsp-api` (container-provided), JSP compiler.

---

## 3. Servlets
- Pure Java classes handling HTTP requests/responses.
- First introduced in late 1990s with Java Servlet API 2.1 (1997).
- Portable across servers supporting the Servlet API:
  - Apache Tomcat
  - Jetty
  - WildFly / JBoss EAP
  - GlassFish / Payara
  - IBM WebSphere
  - Oracle WebLogic
- Clear separation from HTML, but too low-level.
- **Dependencies**: `javax.servlet-api`, servlet container (Tomcat, Jetty).

---

# Servlet-based CRUD APIs

## Old Style (web.xml)

### Folder Structure
```
MyApp/
 ├── src/
 │   └── com/example/HelloServlet.java
 ├── WebContent/
 │   ├── index.html
 │   └── WEB-INF/
 │       └── web.xml
```

### Code

**HelloServlet.java**
```java
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/plain");
        PrintWriter out = response.getWriter();
        out.println("Hello, World (old style)!");
    }
}
```

**web.xml**
```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="3.1">
    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>com.example.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>hello</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```

---

## New Style (@WebServlet)

### Folder Structure
```
MyApp/
 ├── src/
 │   └── com/example/HelloServlet.java
 ├── WebContent/
 │   └── WEB-INF/
 │       └── web.xml (optional/empty)
```

### Code
```java
import java.io.*;
import javax.servlet.*;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/plain");
        PrintWriter out = response.getWriter();
        out.println("Hello, World (new style)!");
    }
}
```

---

# Problems with Servlet-only CRUD APIs

Even in pure Servlet-based apps, you could create separate classes for controller and business logic. But the developer must manually handle many things, which leads to boilerplate and complexity.

**Example of manual separation in old style Servlets:**
```java
public class DiscountController {
    private final DiscountService service = new DiscountService();

    public void handle(HttpServletRequest req, HttpServletResponse res) throws IOException { 
        double discount = service.calculateDiscount(req.getParameter("type"));
        res.getWriter().println(discount);
    }
}

public class DiscountService {
    public double calculateDiscount(String customerType) { 
        // business logic
    }
}
```

✅ Separation exists

❌ But you still have to manually handle:
- Request mapping (/discount)
- JSON serialization
- Error handling
- Dependency injection
- Bean lifecycle

### 1. Layer Separation is Manual in Servlets
- You can separate controller and service, but you manually manage request handling, mapping, serialization, and lifecycle.

### 2. Spring MVC Automates the Boilerplate
- Automatic request mapping via `@RequestMapping` / `@GetMapping` etc., no need to define each URL in `web.xml` as long as component scan is configured in DispatcherServlet.
- Automatic JSON/XML serialization with `@ResponseBody` and Jackson.
- Dependency injection via `@Autowired` / constructor injection.
- Validation & exception handling via annotations like `@Valid` and `@ControllerAdvice`.
- View resolution for JSP/Thymeleaf if needed.
- Testability: Controllers can be tested independently of servlet container using MockMvc.

### 3. Convention + Ecosystem
- Spring MVC gives conventions and a framework: less chance of “spaghetti code” even when multiple developers work on the same app.
- Integrates seamlessly with Spring Boot, Spring Security, Spring Data JPA, etc. — something you would have to wire manually in pure Servlets.

✅ TL;DR:
- You can achieve separation manually in Servlets, but Spring MVC:
  - Reduces boilerplate
  - Handles JSON, validation, exceptions automatically
  - Provides dependency injection and lifecycle management
  - Makes your code more maintainable and testable
  - Integrates easily with the rest of the Spring ecosystem

---

# Spring MVC CRUD API

## With XML (web.xml + dispatcher)
### Folder Structure
```
MyApp/
 ├── src/
 │   └── com/example/controller/HelloController.java
 ├── WebContent/
 │   ├── WEB-INF/
 │   │   ├── web.xml
 │   │   └── dispatcher-servlet.xml
```

### Code

**HelloController.java**
```java
package com.example.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class HelloController {

    @RequestMapping("/hello")
    @ResponseBody
    public String sayHello() {
        return "Hello, World from Spring MVC (XML)!";
    }
}
```

**web.xml**
```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="3.1">
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

**dispatcher-servlet.xml**
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
          http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/context
          http://www.springframework.org/schema/context/spring-context.xsd
          http://www.springframework.org/schema/mvc
          http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <context:component-scan base-package="com.example.controller"/>
    <mvc:annotation-driven/>
</beans>
```

- **Dependencies**:
  - `spring-core`
  - `spring-context`
  - `spring-web`
  - `spring-webmvc`
  - `jackson-databind`

---

## Without XML (Java Config)
### Folder Structure
```
MyApp/
 ├── src/
 │   └── com/example/
 │       ├── AppInitializer.java
 │       ├── WebConfig.java
 │       └── controller/HelloController.java
```

### Code

**HelloController.java**
```java
package com.example.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class HelloController {

    @RequestMapping("/hello")
    @ResponseBody
    public String sayHello() {
        return "Hello, World from Spring MVC (No XML)!";
    }
}
```

**WebConfig.java**
```java
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.example.controller")
public class WebConfig {
    // Extra MVC configuration if needed
}
```

**AppInitializer.java**
```java
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null; 
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

- **Dependencies**: Same as XML-based Spring MVC.

---

# DispatcherServlet & View Resolution

- **DispatcherServlet** ➞ Front Controller in Spring MVC.
- Maps incoming requests to controllers via annotations (`@RequestMapping`).
- Normally, controllers return *view names* (JSPs, Thymeleaf, etc.) ➞ **ViewResolver** maps them.
- With `@ResponseBody`, view resolution is skipped ➞ return value is written directly to HTTP response body (JSON/String).

---

# @ResponseBody
- Normally, a method in a `@Controller` returns a **view name**.
- With `@ResponseBody`, Spring writes the return value **directly to the HTTP response**.
- Used for APIs returning JSON or plain text instead of HTML views.

Example:
```java
@RequestMapping("/hello")
@ResponseBody
public String sayHello() {
    return "Hello";
}
```

---

# How Spring MVC hooks into Tomcat

1. **Servlet API Standard**  
   Tomcat supports `ServletContainerInitializer` (from Servlet 3.0 spec).

2. **Spring’s META-INF/services mechanism**  
   - Spring provides file: `META-INF/services/javax.servlet.ServletContainerInitializer`.  
   - It lists the class: `org.springframework.web.SpringServletContainerInitializer`.  
   - Tomcat reads this file at startup.  
   - `SpringServletContainerInitializer` implements `ServletContainerInitializer`.  
   - Since `ServletContainerInitializer` is part of Servlet spec, Tomcat knows to call it.

3. **SpringServletContainerInitializer Delegation**  
   - Delegates to classes implementing `WebApplicationInitializer`.  
   - Framework developers or users implement `WebApplicationInitializer`.

4. **AbstractAnnotationConfigDispatcherServletInitializer**  
   - A convenience base class for `WebApplicationInitializer`.  
   - Registers **one DispatcherServlet** per implementation.  
   - You can create multiple implementations if you want multiple `DispatcherServlet`s.

5. **DispatcherServlet Creation**  
   - Registers `DispatcherServlet` with mappings (`/` or custom).  
   - Initializes Spring context (`WebConfig` etc.).

---

# Deep Dive into Spring WebMVC

## 1. @Configuration
- Marks class as a **source of Spring bean definitions**.  
- Equivalent to XML `<beans>...</beans>`.  
- Without `@Configuration`, `@Bean` methods won’t be processed properly.  
- If replaced with `@Component`, the class is still picked up, but beans defined via `@Bean` inside it won’t get proper lifecycle management.

## 2. @EnableWebMvc
- Switches on default Spring MVC configuration.  
- Equivalent to `<mvc:annotation-driven/>`.  
- Registers essential beans:
  - `RequestMappingHandlerMapping` ➞ maps URLs to `@RequestMapping` methods.  
  - `RequestMappingHandlerAdapter` ➞ invokes those methods.  
  - `HttpMessageConverters` ➞ handle JSON/XML conversion (via Jackson).  
  - Default config for validation, formatting, etc.

**Without @EnableWebMvc Example:**
```java
@Configuration
@ComponentScan("com.example.controller")
public class WebConfig { }
```
→ Your controllers exist, but Spring won’t map them properly → you’ll get 404 errors.

**With @EnableWebMvc Example:**
```java
@Configuration
@EnableWebMvc
@ComponentScan("com.example.controller")
public class WebConfig { }
```
→ Spring wires controllers, mappings, and JSON serialization.

---
