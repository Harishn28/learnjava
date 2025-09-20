# Evolution of Java Web Development

## 1. CGI (Common Gateway Interface)
- Early way of handling web requests.
- Each request starts a new process → very slow.
- Example: Perl/Python/Java program invoked per request.
- **Dependencies**: None, OS-level executables.

---

## 2. JSP (JavaServer Pages)
- Embeds Java code inside HTML.
- Faster than CGI, but mixes business logic with presentation → messy.
- **Dependencies**: `javax.servlet.jsp-api` (container-provided), JSP compiler.

---

## 3. Servlets
- Pure Java classes handling HTTP requests/responses.
- First introduced in late 1990s with Java Servlet API 2.1 (1997).
- Portable across servers supporting the Servlet API (Tomcat, Jetty, GlassFish, JBoss/WildFly, WebSphere, WebLogic).
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
- Too much boilerplate (request parsing, response writing, JSON serialization manually).
- Hard to test & maintain (tight coupling).
- No clear separation of layers (business logic, controller, view mixed).
- Led to Spring MVC.

1. Layer Separation is Manual in Servlets

Example:
```java
public class DiscountController {
    private final DiscountService service = new DiscountService();

    public void handle(HttpServletRequest req, HttpServletResponse res) throws IOException {
        double discount = service.calculateDiscount(req.getParameter("type"));
        res.getWriter().println(discount);
    }
}

public class DiscountService {
    public double calculateDiscount(String customerType) { ... }
}
```
✅ Separation exists

❌ You still have to manually handle:
- Request mapping (/discount)
- JSON serialization
- Error handling
- Dependency injection
- Bean lifecycle

2. Spring MVC Automates the Boilerplate
- Automatic request mapping via @RequestMapping / @GetMapping etc.
- Automatic JSON/XML serialization with @ResponseBody and Jackson.
- Dependency injection via @Autowired / constructor injection.
- Validation & exception handling via annotations like @Valid and @ControllerAdvice.
- View resolution for JSP/Thymeleaf if needed.
- Testability: Controllers can be tested independently of servlet container using MockMvc.

3. Convention + Ecosystem
- Spring MVC gives conventions and a framework: less chance of “spaghetti code” even when multiple developers work on the same app.
- Integrates seamlessly with Spring Boot, Spring Security, Spring Data JPA, etc.

✅ TL;DR: You can achieve separation manually in Servlets, but Spring MVC:
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

**Note on dispatcher-servlet.xml:**
Even though web.xml does not explicitly reference dispatcher-servlet.xml, Spring automatically looks for a file named `<servlet-name>-servlet.xml` in WEB-INF/ when initializing the DispatcherServlet. In our case, the servlet name is dispatcher, so Spring loads `WEB-INF/dispatcher-servlet.xml` automatically.

- **Dependencies**:
