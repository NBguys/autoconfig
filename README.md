# 스프링 부트의 자동 구성

스프링 부트는 자동 구성(Auto Configuration)이라는 기능을 제공하는데, 일반적으로 자주 사용하는 수 많은 빈들을
자동으로 등록해주는 기능이다.  
스프링 부트는 spring-boot-autoconfigure 라는 프로젝트 안에서 수 많은 자동 구성을 제공한다.  
JdbcTemplate 을 설정하고 빈으로 등록해주는 자동 구성을 확인해보자.  
JdbcTemplateAutoConfiguration

```java
package org.springframework.boot.autoconfigure.jdbc;

import javax.sql.DataSource;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnSingleCandidate;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.sql.init.dependency.DatabaseInitializationDependencyConfigurer;
import org.springframework.context.annotation.Import;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;

@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass({DataSource.class, JdbcTemplate.class})
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(JdbcProperties.class)
@Import({DatabaseInitializationDependencyConfigurer.class,
        JdbcTemplateConfiguration.class,
        NamedParameterJdbcTemplateConfiguration.class})
public class JdbcTemplateAutoConfiguration {
}
```

* @AutoConfiguration : 자동 구성을 사용하려면 이 애노테이션을 등록해야 한다.
    * 자동 구성도 내부에 @Configuration 이 있어서 빈을 등록하는 자바 설정 파일로 사용할 수 있다.
    * after = DataSourceAutoConfiguration.class
        * 자동 구성이 실행되는 순서를 지정할 수 있다. JdbcTemplate 은 DataSource 가 필요하기
          때문에 DataSource 를 자동으로 등록해주는 DataSourceAutoConfiguration 다음에 실행
          하도록 설정되어 있다.
* @ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
    * IF문과 유사한 기능을 제공한다. 이런 클래스가 있는 경우에만 설정이 동작한다. 만약 없으면 여기 있
      는 설정들이 모두 무효화 되고, 빈도 등록되지 않는다.
    * @ConditionalXxx 시리즈가 있다. 자동 구성의 핵심이므로 뒤에서 자세히 알아본다.
    * JdbcTemplate 은 DataSource , JdbcTemplate 라는 클래스가 있어야 동작할 수 있다.
* @Import : 스프링에서 자바 설정을 추가할 때 사용한다.

JdbcTemplateConfiguration

```java
package org.springframework.boot.autoconfigure.jdbc;

import javax.sql.DataSource;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.core.JdbcOperations;
import org.springframework.jdbc.core.JdbcTemplate;

@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(JdbcOperations.class)
class JdbcTemplateConfiguration {
    @Bean
    @Primary
    JdbcTemplate jdbcTemplate(DataSource dataSource, JdbcProperties properties) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        JdbcProperties.Template template = properties.getTemplate();
        jdbcTemplate.setFetchSize(template.getFetchSize());
        jdbcTemplate.setMaxRows(template.getMaxRows());
        if (template.getQueryTimeout() != null) {
            jdbcTemplate.setQueryTimeout((int)
                    template.getQueryTimeout().getSeconds());
        }
        return jdbcTemplate;
    }
}
```
* @Configuration : 자바 설정 파일로 사용된다.
* @ConditionalOnMissingBean(JdbcOperations.class)
  * JdbcOperations 빈이 없을 때 동작한다.
  * JdbcTemplate 의 부모 인터페이스가 바로 JdbcOperations 이다.
  * 쉽게 이야기해서 JdbcTemplate 이 빈으로 등록되어 있지 않은 경우에만 동작한다.
  * 만약 이런 기능이 없으면 내가 등록한 JdbcTemplate 과 자동 구성이 등록하는 JdbcTemplate 이 중복 등록되는 문제가 발생할 수 있다.
  * 보통 개발자가 직접 빈을 등록하면 개발자가 등록한 빈을 사용하고, 자동 구성은 동작하지 않는다. JdbcTemplate 이 몇가지 설정을 거쳐서 빈으로 등록되는 것을 확인할 수 있다.

### 자동 등록 설정
다음과 같은 자동 구성 기능들이 다음 빈들을 등록해준다.
* JdbcTemplateAutoConfiguration : JdbcTemplate
* DataSourceAutoConfiguration : DataSource
* DataSourceTransactionManagerAutoConfiguration : TransactionManager
#### 스프링 부트가 제공하는 자동 구성(AutoConfiguration)
https://docs.spring.io/spring-boot/docs/current/reference/html/auto-configuration-classes.html
* 스프링 부트는 수 많은 자동 구성을 제공하고 spring-boot-autoconfigure 에 자동 구성을 모아둔다.
* 스프링 부트 프로젝트를 사용하면 spring-boot-autoconfigure 라이브러리는 기본적으로 사용된다.

### @Conditional
* 앞서 만든 메모리 조회 기능을 항상 사용하는 것이 아니라 특정 조건일 때만 해당 기능이 활성화 되도록 해 보자.
* 예를 들어서 개발 서버에서 확인 용도로만 해당 기능을 사용하고, 운영 서버에서는 해당 기능을 사용하지 않는 것이다.
* 여기서 핵심은 소스코드를 고치지 않고 이런 것이 가능해야 한다는 점이다.
  * 프로젝트를 빌드해서 나온 빌드 파일을 개발 서버에도 배포하고, 같은 파일을 운영서버에도 배포해야 한다.
* 같은 소스 코드인데 특정 상황일 때만 특정 빈들을 등록해서 사용하도록 도와주는 기능이 바로 @Conditional 이다.
* 참고로 이 기능은 스프링 부트 자동 구성에서 자주 사용한다

#### Condition
```java
package org.springframework.context.annotation;
public interface Condition {
boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
* matches() 메서드가 true 를 반환하면 조건에 만족해서 동작하고, false 를 반환하면 동작하지 않는다.
* ConditionContext : 스프링 컨테이너, 환경 정보등을 담고 있다.
* AnnotatedTypeMetadata : 애노테이션 메타 정보를 담고 있다.

### @ConditionalOnXxx
스프링은 @Conditional 과 관련해서 개발자가 편리하게 사용할 수 있도록 수 많은 @ConditionalOnXxx 를 제공한다.
대표적인 몇가지를 알아보자.
* @ConditionalOnClass , @ConditionalOnMissingClass
  * 클래스가 있는 경우 동작한다. 나머지는 그 반대
* @ConditionalOnBean , @ConditionalOnMissingBean
  * 빈이 등록되어 있는 경우 동작한다. 나머지는 그 반대
* @ConditionalOnProperty
  * 환경 정보가 있는 경우 동작한다.
* @ConditionalOnResource
  * 리소스가 있는 경우 동작한다.
* @ConditionalOnWebApplication , @ConditionalOnNotWebApplication
  * 웹 애플리케이션인 경우 동작한다.
* @ConditionalOnExpression
  * SpEL 표현식에 만족하는 경우 동작한다.

>>> ConditionalOnXxx 공식 메뉴얼
>>> https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.condition-annotations

이름이 직관적이어서 바로 이해가 될 것이다. @ConditionalOnXxx 는 주로 스프링 부트 자동 구성에 사용된다.  
다음 자동 구성 클래스들을 열어서 소스 코드를 확인해보면 @ConditionalOnXxx 가 아주 많이 사용되는 것을 확인  
할 수 있다.  
JdbcTemplateAutoConfiguration , DataSourceTransactionManagerAutoConfiguration ,
DataSourceAutoConfiguration

### 자동 구성 이해1 - 스프링 부트의 동작
스프링 부트는 다음 경로에 있는 파일을 읽어서 스프링 부트 자동 구성으로 사용한다.
>>> resources/META-INF/spring/  
>>> org.springframework.boot.autoconfigure.AutoConfiguration.imports  

우리가 직접 만든 memory-v2 라이브러리와 스프링 부트가 제공하는 spring-boot-autoconfigure 라이브러  
리의 다음 파일을 확인해보면 스프링 부트 자동 구성을 확인할 수 있다.  

memory-v2 - org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
memory.MemoryAutoConfig
```
spring-boot-autoconfigure - org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
...
```

이번에는 스프링 부트가 어떤 방법으로 해당 파일들을 읽어서 동작하는지 알아보자.  
이해를 돕기 위해 앞서 개발한 autoconfig 프로젝트를 열어보자.  
스프링 부트 자동 구성이 동작하는 원리는 다음 순서로 확인할 수 있다.  
@SpringBootApplication -> @EnableAutoConfiguration -> @Import(AutoConfigurationImportSelector.class)  
스프링 부트는 보통 다음과 같은 방법으로 실행한다.  

AutoConfigApplication
```java
@SpringBootApplication
public class AutoConfigApplication {
 public static void main(String[] args) {
 SpringApplication.run(AutoConfigApplication.class, args);
 }
}
```
* run() 에 보면 AutoConfigApplication.class 를 넘겨주는데, 이 클래스를 설정 정보로 사용한다  
  는 뜻이다. AutoConfigApplication 에는 @SpringBootApplication 애노테이션이 있는데, 여기에 중  
  요한 설정 정보들이 들어있다.  

@SpringBootApplication
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes =
TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes =
AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {...} 
```
* 여기서 우리가 주목할 애노테이션은 @EnableAutoConfiguration 이다. 이름 그대로 자동 구성을 활성화 하는 기능을 제공한다.  

@EnableAutoConfiguration
```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {…} 
```
* @Import 는 주로 스프링 설정 정보( @Configuration )를 포함할 때 사용한다.
* 그런데 AutoConfigurationImportSelector 를 열어보면 @Configuration 이 아니다.

### 자동 구성 이해2 - ImportSelector
@Import 에 설정 정보를 추가하는 방법은 2가지가 있다.
* 정적인 방법: @Import (클래스) 이것은 정적이다. 코드에 대상이 딱 박혀 있다. 설정으로 사용할 대상을 동
  적으로 변경할 수 없다.
* 동적인 방법: @Import ( ImportSelector ) 코드로 프로그래밍해서 설정으로 사용할 대상을 동적으로
  선택할 수 있다.

정적인 방법  
* 스프링에서 다른 설정 정보를 추가하고 싶으면 다음과 같이 @Import 를 사용하면 된다. 
```java
@Configuration
@Import({AConfig.class, BConfig.class})
public class AppConfig {...} 
```

동적인 방법  
* 스프링은 설정 정보 대상을 동적으로 선택할 수 있는 ImportSelector 인터페이스를 제공한다.  
ImportSelector
```java
package org.springframework.context.annotation;
public interface ImportSelector {
String[] selectImports(AnnotationMetadata importingClassMetadata);
 //...
}
```

#### @EnableAutoConfiguration 동작 방식
이제 ImportSelector 를 이해했으니 다음 코드를 이해할 수 있다.  
@EnableAutoConfiguration
```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {…} 
```
* AutoConfigurationImportSelector 는 ImportSelector 의 구현체이다. 따라서 설정 정보를
  동적으로 선택할 수 있다.
* 실제로 이 코드는 모든 라이브러리에 있는 다음 경로의 파일을 확인한다.
* META-INF/spring/
  org.springframework.boot.autoconfigure.AutoConfiguration.imports  

memory-v2 - org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
memory.MemoryAutoConfig
```
spring-boot-autoconfigure - org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfigurati
on
...
```
그리고 파일의 내용을 읽어서 설정 정보로 선택한다.  
스프링 부트 자동 구성이 동작하는 방식은 다음 순서로 확인할 수 있다.  
* @SpringBootApplication -> @EnableAutoConfiguration -> @Import(AutoConfigurationImportSelector.class)
* resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 파일을 열어서 설정정보 선택
* 해당 파일의 설정 정보가 스프링 컨테이너에 등록되고 사용

### 정리
스프링 부트의 자동 구성을 직접 만들어서 사용할 때는 다음을 참고하자.
* @AutoConfiguration 에 자동 구성의 순서를 지정할 수 있다.
* @AutoConfiguration 도 설정 파일이다. 내부에 @Configuration 이 있는 것을 확인할 수 있다.
  * 하지만 일반 스프링 설정과 라이프사이클이 다르기 때문에 컴포넌트 스캔의 대상이 되면 안된다.
  * 파일에 지정해서 사용해야 한다.
* resources/META-INF/spring/
  org.springframework.boot.autoconfigure.AutoConfiguration.imports
* 그래서 스프링 부트가 제공하는 컴포넌트 스캔에서는 @AutoConfiguration 을 제외하는
  AutoConfigurationExcludeFilter 필터가 포함되어 있다.  

@SpringBootApplication  
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes =
TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes =
AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {...} 
```
* 자동 구성이 내부에서 컴포넌트 스캔을 사용하면 안된다. 대신에 자동 구성 내부에서 @Import 는 사용할 수 있다