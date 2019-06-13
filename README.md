# spring-boot-tut

## Spring Handler Interceptor

This intercepts after calling the action method and intercepts view creation.

## User of javax.servlet.Filter

Filters are called before actions and can change request's path. To register a filter (Here it is called CustomFilter) we do:

```java
@Bean
public FilterRegistrationBean<LanguageUriFilter> registerLanguageUriFilter(CustomFilter filter) {
    FilterRegistrationBean<LanguageUriFilter> reg = new FilterRegistrationBean<>(filter);
    return reg;
}
```
## Deploy spring boot on a servlet container

From [docs.spring.io](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-create-a-deployable-war-file)

Make main class extends `SpringBootServletInitializer` and override configure method, this way makes use of Spring Framework’s Servlet 3.0 support and lets you configure your application when it is launched by the servlet container.

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(Application.class);
	}

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

In maven, since spring boot's spring-boot-starter-parent maven parent configures maven war plugin, in pom, we only need to change packaging from jar to war.

```maven
<packaging>war</packaging>
```

The final step in the process is to ensure that the embedded servlet container does not interfere with the servlet container to which the war file is deployed. To do so, you need to mark the embedded servlet container dependency as being provided.

Here we assume the servlet container is tomcat, for other containers use their respective servlet providers.
For example for wildfly, use `spring-boot-starter-undertow` 

```maven
<dependencies>
	<!-- … -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-tomcat</artifactId>
		<scope>provided</scope>
	</dependency>
	<!-- … -->
</dependencies>
```
