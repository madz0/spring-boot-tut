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

## Content type `application/x-www-form-urlencoded;charset=UTF-8` not supported

Spring `@RequestBody` only works for `application/json` request. Using it for `application/x-www-form-urlencoded`causes above error.
For `x-www-form-url-encoded` we can use `@RequestParam Map<String, String> body`as the parameter

## Effective hot swap templates

From [so](https://stackoverflow.com/questions/40057057/spring-boot-and-thymeleaf-hot-swap-templates-and-resources-once-again)

Do following changes in properites

```
spring.thymeleaf.cache=false
spring.thymeleaf.mode=LEGACYHTML5
#spring.thymeleaf.prefix=/templates/
spring.thymeleaf.templates_root=src/main/resources/templates/
```

Note that we commnted the `spring.thymeleaf.prefix=/templates/` because we want to always read templates from filesystem.

Now in a configuration class such as main application do:

```java
    @Autowired
    private ThymeleafProperties properties;

    @Value("${spring.thymeleaf.templates_root:}")
    private String templatesRoot;
    @Bean
    public ITemplateResolver defaultTemplateResolver() {
        FileTemplateResolver resolver = new FileTemplateResolver();
        resolver.setSuffix(properties.getSuffix());
        resolver.setPrefix(templatesRoot);
        resolver.setTemplateMode(properties.getMode());
        resolver.setCacheable(properties.isCache());
        return resolver;
    }
```

## Enable internationalization and translated validation

In a configuration do:

```java
@Bean(name = "messageSource")
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
        messageSource.setBasename("classpath:messages");
        messageSource.setFallbackToSystemLocale(false);
        messageSource.setCacheSeconds(0);
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }

    @Bean
    public LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean bean = new LocalValidatorFactoryBean();
        bean.setValidationMessageSource(messageSource());
        return bean;
    }

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver lr = new CookieLocaleResolver();
        lr.setDefaultLocale(DEFAULT_LOCALE);
        return lr;
    }
    ```
    
    Then create translation files inside `resources` using `messages_lang.properties` convention
    For validations put codes inside brackets
