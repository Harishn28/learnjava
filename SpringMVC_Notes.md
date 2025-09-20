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

## Scenario: Building a simple “Discount API"

**Requirements:**

1. GET `/discount?type=VIP` → returns discount percentage.
2. GET `/discount/history` → returns last 5 discounts.
3. Discounts are calculated using business logic (service layer).
4. Output is JSON.
5. Needs proper error handling.
6. Needs to be testable without running a server.

---

## 1️⃣ Plain Servlet Approach

You could do it with two Servlets:

```java
@WebServlet("/discount")
public class DiscountServlet extends HttpServlet {
    private final DiscountService service = new DiscountService();

    protected void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException {
        try {
            String type = req.getParameter("type");
            double discount = service.calculateDiscount(type);

            // Manual JSON
            res.setContentType("application/json");
            res.getWriter().println("{\"discount\":" + discount + "}");
        } catch(Exception e) {
            res.setStatus(500);
            res.getWriter().println("{\"error\":\"Something went wrong\"}");
        }
    }
}

@WebServlet("/discount/history")
public class DiscountHistoryServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException {
        // Manual JSON for list
        res.setContentType("application/json");
        res.getWriter().println("[5, 10, 15, 20]");
    }
}
```

**Problems here:**

1. Each URL requires a **separate Servlet**.
2. Manual JSON serialization.
3. Manual error handling.
4. Hard to inject services or reuse code across Servlets.
5. Testing requires a full Servlet container or complex mocks.

---

## 2️⃣ Spring MVC Approach

Now with **Spring MVC**, we can do **everything in one controller**, with less boilerplate:

```java
@RestController
@RequestMapping("/discount")
public class DiscountController {

    private final DiscountService service;

    public DiscountController(DiscountService service) {
        this.service = service;
    }

    @GetMapping
    public Map<String, Double> getDiscount(@RequestParam String type) {
        double discount = service.calculateDiscount(type);
        return Map.of("discount", discount); // automatic JSON serialization
    }

    @GetMapping("/history")
    public List<Integer> getHistory() {
        return List.of(5, 10, 15, 20); // automatic JSON
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, String>> handleError(Exception e) {
        return ResponseEntity.status(500).body(Map.of("error", e.getMessage()));
    }
}
```

**Advantages:**

1. **One class handles multiple URLs** (`/discount` and `/discount/history`) with **clear annotations**.
2. **Automatic JSON serialization** → no manual string building.
3. **Dependency injection** → no `new DiscountService()` inside the controller.
4. **Centralized error handling** using `@ExceptionHandler`.
5. **Easier testing** → we can test this controller with `MockMvc` **without a Servlet container**.

---

### Key Takeaways

| Feature              | Servlet                            | Spring MVC                                        |
| -------------------- | ---------------------------------- | ------------------------------------------------- |
| URL mapping          | Manual, one servlet per URL        | Annotation-based, multiple methods per controller |
| JSON                 | Manual serialization               | Automatic with Jackson                            |
| Error handling       | Manual                             | Centralized with `@ExceptionHandler`              |
| Dependency injection | Manual                             | `@Autowired` or constructor injection             |
| Testability          | Hard                               | Easy with `MockMvc`                               |
| Scalability          | Many servlets, lots of boilerplate | One controller handles multiple endpoints cleanly |

---

✅ **Bottom line:**

Yes, you **can** achieve separation and mapping manually with Servlets, but Spring MVC **automates repetitive tasks**, **reduces boilerplate**, and integrates **dependency injection, validation, error handling, and testing**. This makes building and maintaining complex apps feasible.





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

*Note on dispatcher-servlet.xml:
Even though web.xml does not explicitly reference dispatcher-servlet.xml, Spring automatically looks for a file named <servlet-name>-servlet.xml in WEB-INF/ when initializing the DispatcherServlet. In our case, the servlet name is dispatcher, so Spring loads WEB-INF/dispatcher-servlet.xml automatically.*
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
------
# How Spring MVC Hooks into Tomcat (Explained)

This explains step by step how Spring MVC initializes within a servlet container like Tomcat.

---

## 1️⃣ Servlet API Standard

* Starting from **Servlet 3.0**, servlet containers like **Tomcat** introduced a mechanism called `ServletContainerInitializer`.
* This is a **hook**: the container says, “If someone wants to run code when the server starts, I’ll call this interface for them.”

---

## 2️⃣ Spring’s META-INF/services Mechanism

* Spring creates the file:

  ```
  META-INF/services/javax.servlet.ServletContainerInitializer
  ```
* Inside it lists the class:

  ```
  org.springframework.web.SpringServletContainerInitializer
  ```
* Tomcat reads this file automatically at startup and calls Spring’s initializer.

**Think of it as:**

> Tomcat says: “Who wants to do something at startup?”
> Spring says: “We do!”
> Tomcat calls Spring’s initializer.

---

## 3️⃣ SpringServletContainerInitializer Delegation

* `SpringServletContainerInitializer` itself doesn’t configure your app.
* It **looks for classes that implement `WebApplicationInitializer`**.
* Your custom initialization logic is handled by these classes.

---

## 4️⃣ AbstractAnnotationConfigDispatcherServletInitializer

* Spring provides this base class to simplify `WebApplicationInitializer`.
* What it does:

  * Registers **DispatcherServlet**.
  * Loads Spring context (`@Configuration` classes).
  * Maps URLs (default `/`).

**Think of it as:** A helper that wires everything automatically.

---

## 5️⃣ DispatcherServlet Creation

* DispatcherServlet = Spring MVC **front controller**.
* On startup:

  1. Registered with the servlet container.
  2. URL mappings set up (`/` or custom).
  3. Spring context loads controllers, `@RequestMapping`, and beans.

---

## Diagram: Flow from Tomcat Startup → Spring MVC

```
Tomcat Startup
      |
      v
ServletContainerInitializer (Servlet 3.0 Spec)
      |
      v
SpringServletContainerInitializer (Spring)
      |
      v
WebApplicationInitializer (User / AbstractAnnotationConfigDispatcherServletInitializer)
      |
      v
Registers DispatcherServlet + Loads Spring Context
      |
      v
Controllers (@Controller) and RequestMappings (@RequestMapping) ready
      |
      v
HTTP Requests --> DispatcherServlet --> Controllers --> Response
```

---

## ✅ TL;DR

1. Tomcat provides startup hook (`ServletContainerInitializer`).
2. Spring registers its initializer (`SpringServletContainerInitializer`).
3. Spring finds your `WebApplicationInitializer`.
4. DispatcherServlet is registered and Spring context is loaded.
5. Spring MVC is ready to handle requests.


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
