# spring-boot-tut

-- Spring Handler Interceptor

This intercepts after calling the action method and intercepts view creation.

-- User of javax.servlet.Filter

Filters are called before actions and can change request's path. To register a filter (Here it is called CustomFilter) we do:

```
@Bean
    public FilterRegistrationBean<LanguageUriFilter> registerLanguageUriFilter(CustomFilter filter) {
        FilterRegistrationBean<LanguageUriFilter> reg = new FilterRegistrationBean<>(filter);
        return reg;
    }
```
