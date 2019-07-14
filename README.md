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
 
## Configurations and Starters design for better testing tip
To make the software testable with faster tests, *devide configuaraion and starter components into smaller classes*. 
 
In integration testing, `@SpringBootTest` is going to bring every thing up. So it makes testing so slow. 
Everything is devided into the following parts:
 
Web layer, Data layer, Application Context layer
 
Both the web and data needs application context.
When we use @SpringBootTest, it brings up all the three layers and it also brings up all the components for application context and this is the main reason why the tests are so slow in larger applications.

To make it better, we should act differently for each test or at least have different base classes for different tests. But before going any further, lets see how can we elliminate the @SpringBootTest

If we are going to test a controller, we only need the web layer and all application context for some limited components.
So we use `@WebMvcTest` or its equivallent in reactive `@WebFluxTest`:
```java
@RunWith(SpringRunner.class)
@WebMvcTest(EmployeeRestController.class)
public class EmployeeRestControllerIntegrationTest{
//test method
}
```
Then we can use `@MockBean` to mock service layers. In an application with lots of bean deffinition, this would dramatically increase the run speed.

For testing the sevice layer itself, I don't think we need any integration test, so simply can use `@MockBean` but `@MockBean` needs application context.
To elliminate that, we can use of `@TestConfiguration` as a inner static class inside the test:
```java
@RunWith(SpringRunner.class)
public class EmployeeServiceImplIntegrationTest {
 
    @TestConfiguration
    static class EmployeeServiceImplTestContextConfiguration {
  
        @Bean
        public EmployeeService employeeService() {
            return new EmployeeServiceImpl();
        }
    }
    
    //use @Before
 
    @Autowired
    private EmployeeService employeeService;
 
    @MockBean
    private EmployeeRepository employeeRepository;
 
    // write test cases here
}
```
This way we don't need to bring up application context
*During component scanning, we might find components or configurations created only for specific tests accidentally get picked up everywhere. To help prevent that, Spring Boot provides @TestConfiguration annotation that can be used on classes in src/test/java to indicate that they should not be picked up by scanning.*

If we intended to test the data layer, we can use `@DataJpaTest` 

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class EmployeeRepositoryIntegrationTest {
 
    @Autowired
    private TestEntityManager entityManager;
 
    @Autowired
    private EmployeeRepository employeeRepository;
 
    // write test cases here
}
```
`@RunWith(SpringRunner.class)` is used to provide a bridge between Spring Boot test features and JUnit. Whenever we are using any Spring Boot testing features in out JUnit tests, this annotation will be required.

`@DataJpaTest` provides some standard setup needed for testing the persistence layer:

* configuring `H2`, an in-memory database
* setting `Hibernate`, `Spring Data`, and the `DataSource`
* performing an `@EntityScan`
* turning on `SQL` logging

To carry out some DB operation, we need some records already setup in our database. To setup such data, we can use TestEntityManager. The `TestEntityManager` provided by Spring Boot is an alternative to the standard `JPA EntityManager` that provides methods commonly used when writing tests.

[spring-boot-testing](https://www.baeldung.com/spring-boot-testing)

As an example consider that we are going to do an integration test that uses a bean to do something. for this test, we only need to utilize autowired feature so we need only to bring up context and don't need web layer or data layer.
If we use @SpringBootTest, it would bring up every thing. So we are going to use `@ContextConfiguration(classes = .{..})`
But here, the classes is so important. If we use the main application class, again it would bring up so many beans because of the @SpringBootApplication (ComponentScan will be executed). 
So to bring the smallest bean possible, we only put the direct Bean class and its dependencies in `classes` parameter.

In the following example:

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = {MibRepositoryCached.class, MibbleMibCompiler.class, PathMatchingResourcePatternResolver.class})
@TestPropertySource(locations = "classpath:application.properties")
public class ExampleTest {
    @Autowired
    IMibRepository mibRepository;
    @Autowired
    ApplicationContext context;

    @Before
    public void init() throws BadArgumentException {
        MainApplication coreApplication = new MainApplication();
        coreApplication.setApplicationContext(context);
    }

    @Test
    public void test() {
    }
}
```

here we've bring the `MibRepositoryCached` class that is an implementation of IMibRepository. This way the context would create that bean and autowired will work. But `MibRepositoryCached` has other dependencies to other beans called `MibbleMibCompiler` and `PathMatchingResourcePatternResolver` which make `MibRepositoryCached` failed. So we bring them too.

The other autowired here is the `ApplicationContext` which is going to be created and injected by @ContextConfiguration.
The reason to do that is because the main application is implemented `ApplicationContextAware` and defined as following:

```java
@Slf4j
@SpringBootApplication
@EnableScheduling
@EnableCaching
@EnableJpaRepositories(repositoryFactoryBeanClass = EntityGraphJpaRepositoryFactoryBean.class)
public class MainApplication implements ApplicationContextAware {
	private static ApplicationContext context;
	public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
    
    @Override
    public ApplicationContext getApplicationContext() {
        return context;
    }
    
    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }
}
```

We use `MainApplication.getBean()` in a places related to this test but since we don't bring up MainApplication, context is not be assigned. So we use @Before to set the context ourselves. This way the code is much much faster

## Map native query result to entity and dto objects

If the returned columns of the native query maches target entity, it implicitely converts it to entity
Otherwise, define the mapping on top of the entity class using

```java
@SqlResultSetMapping(
  name="EmployeeResult",
  entities={
    @EntityResult(
      entityClass = Employee.class,
        fields={
          @FieldResult(name="id",column="employeeNumber"),
          @FieldResult(name="name", column="name")})})
@NamedNativeQuery(name = "findAllDataMapping", resultClass = Employee.class, resultSetMapping ="findAllDataMapping" query = "sql")
```
and use something like this:
```java
@Query(nativeQuery = true, name = "findAllDataMapping")
```
Or we can use `Tuple` data type

## Spring tarnsactional

1. Spring `@Transactional` only rollbacks for unchecked exception. So in the following example:

	```java
	@Transactional
	public void someMethodInTransaction() {
		okMethod();
		notOkMethod()
	}

	void okMethod() {
	//some work
	}

	void notOkMethod() throws Exception {
		throw new Exception();	
	}
	```
	when running `someMethodInTransaction` `okMethod` wold do its job event if `notOkMethod` throws `Exception`

2. In the followong example, 
	```java
	@Transactional
	public void changeName(long id, String name) {
	 User user = userRepository.getById(id);
	 user.setName(name);
	 userRepository.save(user);
	}
	```
	Either `@Transactional` or `userRepository.save(user)` is redundant. Because in a transactional scope every change to manage objects will be presisted at the end of transaction scope. Also the following use is wrong:
	```java
	@Transactional
	public void changeName(long id, String name) {
	  User user = userRepository.getById(id);
	  user.setName(name);

	  if (StringUtils.isNotEmpty(name)) {
	    userRepository.save(user);
	  }
	}
	```
	`user.setName(name)` should go inside the validation check of event better its better to check validation befor transaction scope

3. An important thing to remember is that readOnly hint will be applied only when the corresponding @Transactionalcauses starting a completely new transaction.

4. Any change made within a transaction to an entity (retrieved within the same transaction) will automatically be populated to the database at the end of the transaction, without the need of explicit manual updates.
