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

4. Any change made within a transaction to an entity (retrieved within the same transaction) will automatically be populated to the database at the end of the sql first n rows with bigger valuestransaction, without the need of explicit manual updates.

## Getting `DataIntegrationException` when using `@JoinColumn` with foreign key definition

Sometimes you get `DataIntegrationException` when using `@JoinColumn` with foreign key definition and when you remove joincolumn, everything works. In that case, there is something wrong about data types in both sides. For example the referenced column may be a datetime(6) but refencing column is a datetime(20)

## @Lob in postgresql explanation

A lob is a large object. Lob columns can be used to store very long texts or binary files. There are two kind of lobs: CLOB and BLOB. The first one is a character lob and can be used to store texts. It is an alternative to varchar, which is limited in most databases. The second is a binary lob and can be used to store binary files.

A character lob is very simple, if you use PostgreSQL. In the table it is represented as text and can take any size of characters:

```sql
CREATE TABLE annotation.document
(
  largeText text,
  ...
```

The class has a simple String field:
```java
private String text;
```

Annotation mapping. 
```java
import javax.persistence.Entity;
import javax.persistence.Lob;
..... snip ......
@Entity
public class Document implements Serializable {
...... snip .......
   @Lob
   private String text;
```

Just add a @Lob annotation to a string field. That’s all

### Binary lob (blob)

There are two options to store blobs in PostgreSQL. The first approach stores the file directly in the column. The type of such a column is bytea, which is a short form of byte array. The second approach is to store a OID in the column which references a file. PostgreSQL keeps the file separately from your table. This type of such a column is blob or binary object. You can store large blobs in both column types. The simpler way is to use bytea, but PostgreSQL needs a lot of memory, if you use the bytea column PostgreSQL and select a lot of rows having large bytea columns. Columns of type blob or binaryobject can read the lob as stream, once you access the data. The disadvantage of blob/binaryobject is that if you delete a table row the file is not deleted automatically. The following examples will show a work around for this problem. So you may freely select any of this approaches.

The PostgreSQL table is having a bytea and a blob column.
```sql
CREATE TABLE annotation.image
(
  imageasblob oid,
  imageasbytea bytea,
......
```

The bytea approach needs a byte array field in the class. I used imageasBytea[] Annotation mapping: It is important that you specify the type. You will get a blob column, if you don’t.

```java
import javax.persistence.Entity;
import org.hibernate.annotations.Type;
...... snip ......
@Entity
public class Image implements Serializable {

   @Type(type = "org.hibernate.type.BinaryType")
   private byte imageAsBytea[];
........
```
If your field is of type byte[] you don’t have to specify a type. We added it optionally. There should be not @Lob for the field to be saved as bytea.

Samples of use. 
```java
/* create a byte array and set it */
byte byteArray[] = new byte[10000000];
for (int i = 0; i < byteArray.length; i++) {
   byteArray[i] = '1';
}
Image image = new Image();
image.setImageAsBytea(byteArray);
/* write a field to a file */
FileOutputStream outputStream =
   new FileOutputStream(new File("image_file_bytea"));
outputStream.write(image.getImageAsBytea());
```

The blob approach requires some additional code. If we delete or update a blob image, the large object will not be deleted as well. Therefore we add a rule to the database, which will provide this for us. Hibernate will not see any of this code, but can simply rely on the fact that, if it deletes an entry. the lob will be deleted as well. Tip: The large object file will not be deleted automatically, if you use a blob column. In the psql client or pgadmin issue the following two statements. They will create rules. The first rule is called when a row is deleted. It deletes the corresponding lob. The second rule is called, when a row is updated. It deletes the old image, if the image has changed.
```sql
CREATE RULE droppicture AS ON DELETE TO annotation.image
  DO SELECT lo_unlink( OLD.imageasblob );
CREATE RULE reppicture AS ON UPDATE TO annotation.image
  DO SELECT lo_unlink( OLD.imageasblob )
   where OLD.imageasblob <> NEW.imageasblob;
```
We have two options in the class: a java.sql.Blob field and a byte array. We can use a byte array to map a lob to a blob column as well.

Annotation mapping. 
```java
import java.sql.Blob;
import javax.persistence.Entity;
import javax.persistence.Lob;
...... snip .........
import org.hibernate.annotations.Type;
import org.hibernate.type.BlobType;@Entity
public class Image implements Serializable {
   @Lob
   private byte imageAsBlob[];
   private Blob imageAsBlob2;
```
Samples of use. 

```java
/* creating a blob */
byte byteArray[] = new byte[10000000];
for (int i = 0; i < byteArray.length; i++) {
   byteArray[i] = '1';
}
Image image = new Image();
image.setImageAsBlob(byteArray);  // a blob as byte array
image.setImageAsBlob2(Hibernate.createBlob(byteArray)); // a blob as blob
/* reading */
// read blob from a byte array is as simple as from a bytea
FileOutputStream outputStream =
   new FileOutputStream(new File("image_file_blob_array"));
outputStream.write(image.getImageAsBlob());
outputStream.close();
// reading of a blob from a blob is in fact a inputstream
outputStream = new FileOutputStream(new File("image_file_blob_blob"));
outputStream.write(image.getImageAsBlob2()
   .getBytes(1,(int)image.getImageAsBlob2().length()));
outputStream.close();
```
Tip: You can only access the length field if your transaction is open.
```java
image.getImageAsBlob2().length()
```
Tip: 
reference [http://www.laliluna.de/jpa-hibernate-guide/ch10.html](http://www.laliluna.de/jpa-hibernate-guide/ch10.html)

## Cascading consists in propagating the Parent entity state transition to one or more Child entities, and it can be used for both unidirectional and bidirectional associations.

## Try not to use CascadeType especially remove for @ManyToMany association

## Great article about @OneToMany mapping

[https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/)

## Cautions to consider when mapping multiple entities to the same table 
[https://thoughts-on-java.org/hibernate-tips-map-multiple-entities-same-table/](https://thoughts-on-java.org/hibernate-tips-map-multiple-entities-same-table/)
### We can map an entity with `@Immutable` annotation this way hibernate does not use it update table (a proper way to map a view) 

## In the case of assigned identity for jpa, spring save uses following code:

```java
@Transactional
public <S extends T> S save(S entity) {
 
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```
So if we assign value to id, merge will be executed. merge always issue a select to database. If it does not find any record, executes an insert.
To prevent this, we can use a @Version property and assign null to it. This way merge will not execute a select before insert.

## Best way to have a compound identity generator and assigned id in hibernate (https://vladmihalcea.com/how-to-combine-the-hibernate-assigned-generator-with-a-sequence-or-an-identity-column/)[https://vladmihalcea.com/how-to-combine-the-hibernate-assigned-generator-with-a-sequence-or-an-identity-column/]

## Define custom types for hibernate

Recently in a project we needed to use postgresql jsonb type. We used a library called hibernate-types and it worked. 
But all our test failed because H2 did not support jsonb. 
The problem was that in our entity we used `@Column(columnDefinition="jsonb")` and hibernate tried creating a table with `jsonb` type.

I needed a way to somehow alias the jsonb to text for the H2 database.
First I came up with this code 
```sql
CREATE domain IF NOT EXISTS jsonb AS other;
```
I decided to run this query before hibernate creates the tables. To my surprise, i could not find a way to execute a sql code before hibernate generates database tables! 
For example there were a way to create a `schema.sql` in `resources` and spring would execute them but it won't work a long side with hibernate `ddl-auto` other than `none` or `validate`.

So I could not find out if that solution works or not.
I decided to use custom types which maps `jsonb` to `jsonb` type in production and `text` type in tests.

In `hibernate`, there are two ways to define a customtype. By implementing `UserType` interface or by extending a `AbstractSingleColumnStandardBasicType` class.
`UserType` method is somehow strait forward. 
We only need to implement its methods then register it in some place like a base entity or packa-info file using `@TypeDef` annotation.
We also need to extend database `dialect` class and `registerColumnType` for the for the `jsonb`. 
Then change database properties to use the extended `dielect`.

There is a good sample explained in (https://thoughts-on-java.org/persist-postgresqls-jsonb-data-type-hibernate/)[https://thoughts-on-java.org/persist-postgresqls-jsonb-data-type-hibernate/]

For the second way, we don't need to create new dialectic but we need to do a few other things.
When extending `AbstractSingleColumnStandardBasicType`, there is a method called `getName()`.
We need to override it and return the database column type we want to support. In this case `jsonb`.
Its constructor takes two parameters. 
First a `sqlTypeDescriptor` and then the corresponding `javaTypeDescriptor`
The java type descriptor extends `AbstractTypeDescriptor` and `sqlTypeDescriptor` implements `SqlTypeDescriptor`.

`SqlTypeDescriptor` has a `getSqlType()` method which we use to return the supported sql type by hibernate. This method does the `registerColumnType` method we used in `dialect` for the previous method. 
Then we just need to register extended `AbstractSingleColumnStandardBasicType` using `@TypeDef` just like we did in the first way.

Using above information, I created a package-info file in the methods package and put the following contents in there:
```java
@org.hibernate.annotations.TypeDef(name = "jsonb", typeClass = JsonBinaryType.class)
package com.ourproject.model;

import com.vladmihalcea.hibernate.type.json.JsonBinaryType;
```
Then in the entity class, I removed `columnDefinition="jsonb"` from the `@Column` and only used `@Type(type = "jsonb")`

This way, for the product hibernate would map column to `jsonb` type.
Then in the test folder inside the same package I added `package-info` with this contents:

```java
@org.hibernate.annotations.TypeDef(name = "jsonb", typeClass = TextType.class)
package com.ourproject.model;

import org.hibernate.type.TextType;
```
Now when we run maven test hibernate generates `text` column types for the jsonb type and it solved. 
