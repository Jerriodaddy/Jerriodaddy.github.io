---
title: Shrio VS Spring Security [转]
date: 2021-06-23 17:20:57
tags: [SpringBoot, Security]
category: System Architecture
---
Security is a primary concern in the world of application development, especially in the area of enterprise web and mobile applications.

In this quick tutorial, **we'll compare two popular Java Security frameworks – [Apache Shiro](https://shiro.apache.org/) and [Spring Security](https://spring.io/projects/spring-security).**
<!-- more -->

## 1. Overview
Security is a primary concern in the world of application development, especially in the area of enterprise web and mobile applications.

In this quick tutorial, **we'll compare two popular Java Security frameworks – [Apache Shiro](https://shiro.apache.org/) and [Spring Security](https://spring.io/projects/spring-security).**

## 2. A Little Background
Apache Shiro was born in 2004 as JSecurity and was accepted by the Apache Foundation in 2008. To date, it has seen many releases, the latest as of writing this is 1.5.3.

Spring Security started as Acegi in 2003 and was incorporated into the Spring Framework with its first public release in 2008. Since its inception, it has gone through several iterations and the current GA version as of writing this is 5.3.2.

Both technologies offer **authentication and authorization support along with cryptography and session management solutions**. Additionally, Spring Security provides first-class protection against attacks such as CSRF and session fixation.

In the next few sections, we'll see examples of how the two technologies handle authentication and authorization. To keep things simple, we'll be using basic Spring Boot based MVC applications with FreeMarker templates.

## 3. Configuring Apache Shiro
To start with, let's see how configurations differ between the two frameworks.

### 3.1. Maven Dependencies
Since we'll use Shiro in a Spring Boot App, we'll need its starter and the shiro-core module:
```java
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring-boot-web-starter</artifactId>
    <version>1.5.3</version>
</dependency>
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-core</artifactId>
    <version>1.5.3</version>
</dependency>
```
The latest versions can be found on Maven Central.

### 3.2. Creating a Realm
To declare users with their roles and permissions in-memory, we need to create a realm extending Shiro's JdbcRealm. We'll define two users – Tom and Jerry, with roles USER and ADMIN, respectively:
```java
public class CustomRealm extends JdbcRealm {

    private Map<String, String> credentials = new HashMap<>();
    private Map<String, Set> roles = new HashMap<>();
    private Map<String, Set> permissions = new HashMap<>();

    {
        credentials.put("Tom", "password");
        credentials.put("Jerry", "password");

        roles.put("Jerry", new HashSet<>(Arrays.asList("ADMIN")));
        roles.put("Tom", new HashSet<>(Arrays.asList("USER")));

        permissions.put("ADMIN", new HashSet<>(Arrays.asList("READ", "WRITE")));
        permissions.put("USER", new HashSet<>(Arrays.asList("READ")));
    }
}
```
Next, to enable retrieval of this authentication and authorization, we need to override a few methods:
```java
@Override
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) 
  throws AuthenticationException {
    UsernamePasswordToken userToken = (UsernamePasswordToken) token;

    if (userToken.getUsername() == null || userToken.getUsername().isEmpty() ||
      !credentials.containsKey(userToken.getUsername())) {
        throw new UnknownAccountException("User doesn't exist");
    }
    return new SimpleAuthenticationInfo(userToken.getUsername(), 
      credentials.get(userToken.getUsername()), getName());
}

@Override
protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
    Set roles = new HashSet<>();
    Set permissions = new HashSet<>();

    for (Object user : principals) {
        try {
            roles.addAll(getRoleNamesForUser(null, (String) user));
            permissions.addAll(getPermissions(null, null, roles));
        } catch (SQLException e) {
            logger.error(e.getMessage());
        }
    }
    SimpleAuthorizationInfo authInfo = new SimpleAuthorizationInfo(roles);
    authInfo.setStringPermissions(permissions);
    return authInfo;
}
```
The method *doGetAuthorizationInfo* is using a couple of helper methods to get the user's roles and permissions:
```java
@Override
protected Set getRoleNamesForUser(Connection conn, String username) 
  throws SQLException {
    if (!roles.containsKey(username)) {
        throw new SQLException("User doesn't exist");
    }
    return roles.get(username);
}

@Override
protected Set getPermissions(Connection conn, String username, Collection roles) 
  throws SQLException {
    Set userPermissions = new HashSet<>();
    for (String role : roles) {
        if (!permissions.containsKey(role)) {
            throw new SQLException("Role doesn't exist");
        }
        userPermissions.addAll(permissions.get(role));
    }
    return userPermissions;
}
```
Next, we need to include this *CustomRealm* as a bean in our Boot Application:

```java
@Bean
public Realm customRealm() {
    return new CustomRealm();
}
```
Additionally, to configure authentication for our endpoints, we need another bean:

```java
@Bean
public ShiroFilterChainDefinition shiroFilterChainDefinition() {
    DefaultShiroFilterChainDefinition filter = new DefaultShiroFilterChainDefinition();

    filter.addPathDefinition("/home", "authc");
    filter.addPathDefinition("/**", "anon");
    return filter;
}
```
Here, using a *DefaultShiroFilterChainDefinition* instance, we specified that our */home* endpoint can only be accessed by authenticated users.

That's all we need for the configuration, Shiro does the rest for us.

## 4. Configuring Spring Security
Now let's see how to achieve the same in Spring.

### 4.1. Maven Dependencies
First, the dependencies:
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
The latest versions can be found on [Maven Central](https://search.maven.org/classic/#search%7Cga%7C1%7Cg%3A%22org.springframework.boot%22%20AND%20a%3A%22spring-boot-starter%22).

### 4.2. Configuration Class
Next, we'll define our Spring Security configuration in a class *SecurityConfig*, extending *WebSecurityConfigurerAdapter*:
```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
          .authorizeRequests(authorize -> authorize
            .antMatchers("/index", "/login").permitAll()
            .antMatchers("/home", "/logout").authenticated()
            .antMatchers("/admin/**").hasRole("ADMIN"))
          .formLogin(formLogin -> formLogin
            .loginPage("/login")
            .failureUrl("/login-error"));
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
          .withUser("Jerry")
            .password(passwordEncoder().encode("password"))
            .authorities("READ", "WRITE")
            .roles("ADMIN")
            .and()
          .withUser("Tom")
            .password(passwordEncoder().encode("password"))
            .authorities("READ")
            .roles("USER");
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
As we can see, we built an *AuthenticationManagerBuilder* object to declare our users with their roles and authorities. Additionally, we encoded the passwords using a *BCryptPasswordEncoder*.

Spring Security also provides us with its *HttpSecurity* object for further configurations. For our example, we've allowed:
- everyone to access our *index* and *login* pages
- only authenticated users to enter the *home* page and *logout*
- only users with ADMIN role to access the *admin* pages

We've also defined support for form-based authentication to send users to the *login* endpoint. In case login fails, our users will be redirected to */login-error.*

## 5. Controllers and Endpoints
Now let's have a look at our web controller mappings for the two applications. While they'll use the same endpoints, some implementations will differ.

### 5.1. Endpoints for View Rendering
For endpoints rendering the view, the implementations are the same:
```java
@GetMapping("/")
public String index() {
    return "index";
}

@GetMapping("/login")
public String showLoginPage() {
    return "login";
}

@GetMapping("/home")
public String getMeHome(Model model) {
    addUserAttributes(model);
    return "home";
}
```
Both our controller implementations, Shiro as well as Spring Security, return the *index.ftl* on the root endpoint, *login.ftl* on the login endpoint, and *home.ftl* on the home endpoint.

However, the definition of the method *addUserAttributes* at the */home* endpoint will differ between the two controllers. This method introspects the currently logged in user's attributes.

Shiro provides a *SecurityUtils.getSubject* to retrieve the current *Subject*, and its roles and permissions:
```java
private void addUserAttributes(Model model) {
    Subject currentUser = SecurityUtils.getSubject();
    String permission = "";

    if (currentUser.hasRole("ADMIN")) {
        model.addAttribute("role", "ADMIN");
    } else if (currentUser.hasRole("USER")) {
        model.addAttribute("role", "USER");
    }
    if (currentUser.isPermitted("READ")) {
        permission = permission + " READ";
    }
    if (currentUser.isPermitted("WRITE")) {
        permission = permission + " WRITE";
    }
    model.addAttribute("username", currentUser.getPrincipal());
    model.addAttribute("permission", permission);
}
```
On the other hand, Spring Security provides an *Authentication* object from its *SecurityContextHolder‘s* context for this purpose:

```java
private void addUserAttributes(Model model) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null && !auth.getClass().equals(AnonymousAuthenticationToken.class)) {
        User user = (User) auth.getPrincipal();
        model.addAttribute("username", user.getUsername());
        Collection<GrantedAuthority> authorities = user.getAuthorities();

        for (GrantedAuthority authority : authorities) {
            if (authority.getAuthority().contains("USER")) {
                model.addAttribute("role", "USER");
                model.addAttribute("permissions", "READ");
            } else if (authority.getAuthority().contains("ADMIN")) {
                model.addAttribute("role", "ADMIN");
                model.addAttribute("permissions", "READ WRITE");
            }
        }
    }
}
```
### 5.2. POST Login Endpoint
In Shiro, we map the credentials the user enters to a POJO:
```java
public class UserCredentials {

    private String username;
    private String password;

    // getters and setters
}
```
Then we'll create a *UsernamePasswordToken* to log the user, or *Subject*, in:
```java
@PostMapping("/login")
public String doLogin(HttpServletRequest req, UserCredentials credentials, RedirectAttributes attr) {

    Subject subject = SecurityUtils.getSubject();
    if (!subject.isAuthenticated()) {
        UsernamePasswordToken token = new UsernamePasswordToken(credentials.getUsername(),
          credentials.getPassword());
        try {
            subject.login(token);
        } catch (AuthenticationException ae) {
            logger.error(ae.getMessage());
            attr.addFlashAttribute("error", "Invalid Credentials");
            return "redirect:/login";
        }
    }
    return "redirect:/home";
}
```
On the Spring Security side, this is just a matter of redirection to the home page. Spring's logging-in process, handled by its *UsernamePasswordAuthenticationFilter*, is transparent to us:
```java
@PostMapping("/login")
public String doLogin(HttpServletRequest req) {
    return "redirect:/home";
}
```

### 5.3. Admin-Only Endpoint
Now let's look at a scenario where we have to perform role-based access. Let's say we have an */admin* endpoint, access to which should only be allowed for the ADMIN role.

Let's see how to do this in Shiro:
```java
@GetMapping("/admin")
public String adminOnly(ModelMap modelMap) {
    addUserAttributes(modelMap);
    Subject currentUser = SecurityUtils.getSubject();
    if (currentUser.hasRole("ADMIN")) {
        modelMap.addAttribute("adminContent", "only admin can view this");
    }
    return "home";
}
```
Here we extracted the currently logged in user, checked if they have the ADMIN role, and added content accordingly.

In Spring Security, there is no need for checking the role programmatically, we've already defined who can reach this endpoint in our *SecurityConfig*. So now, it's just a matter of adding business logic:
```java
@GetMapping("/admin")
public String adminOnly(HttpServletRequest req, Model model) {
    addUserAttributes(model);
    model.addAttribute("adminContent", "only admin can view this");
    return "home";
}
```

### 5.4. Logout Endpoint
Finally, let's implement the logout endpoint.

In Shiro, we'll simply call *Subject.logout*:
```java
@PostMapping("/logout")
public String logout() {
    Subject subject = SecurityUtils.getSubject();
    subject.logout();
    return "redirect:/";
}
```
For Spring, we've not defined any mapping for logout. In this case, its default logout mechanism kicks in, which is automatically applied since we extended *WebSecurityConfigurerAdapter* in our configuration.

## 6. Apache Shiro vs Spring Security
Now that we've looked at the implementation differences, let's look at a few other aspects.

In terms of community support, the **Spring Framework in general has a huge community of developers**, actively involved in its development and usage. Since Spring Security is part of the umbrella, it must enjoy the same advantages. Shiro, though popular, does not have such humongous support.

Concerning documentation, Spring again is the winner.

However, there's a bit of a learning curve associated with Spring Security. **Shiro, on the other hand, is easy to understand**. For desktop applications, configuration via [shiro.ini](https://shiro.apache.org/configuration.html#Configuration-CreatingaSecurityManagerfromINI) is all the easier.

But again, as we saw in our example snippets, **Spring Security does a great job of keeping business logic and security separate** and truly offers security as a cross-cutting concern.

## 7. Conclusion
In this tutorial, **we compared Apache Shiro with Spring Security**.

We've just grazed the surface of what these frameworks have to offer and there is a lot to explore further. There are quite a few alternatives out there such as [JAAS](https://docs.oracle.com/en/java/javase/11/security/java-authentication-and-authorization-service-jaas-reference-guide.html#GUID-2A935F5E-0803-411D-B6BC-F8C64D01A25C) and [OACC](http://oaccframework.org/). Still, with its advantages, [Spring Security](https://www.baeldung.com/security-spring) seems to be winning at this point.

As always, source code is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/apache-shiro). 

--------
[转自][Spring Security vs Apache Shiro](https://www.baeldung.com/spring-security-vs-apache-shiro)
Author: Sampada Wagde
Access date: Jun 23, 2021

