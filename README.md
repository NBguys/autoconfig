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

