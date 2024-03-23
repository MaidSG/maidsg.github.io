---

layout: post

title: "2024-03-23-Spring SpEL"

date:   2024-03-23

tags: [coding,springframework note,basic knowledge]

comments: true

author: 阿昱

---

SpEL(Spring Expression Language)，即Spring表达式语言。它是一种类似JSP的EL表达式、但又比后者更为强大有用的表达式语言。

SpEL 通常用于以下场景：

## **Spring 配置文件**：在 Bean 定义中使用 SpEL，在运行时动态地注入值
```xml
<bean id="city" class="com.example.City">
    <property name="name" value="New York"/>
</bean>

<bean id="person" class="com.example.Person">
    <property name="city" ref="city"/>
    <property name="name" value="#{city.name}"/>
</bean>
```

在上述配置中，#{city.name}就是一个SpEL表达式。当Spring容器初始化person bean时，它会解析这个表达式，获取city bean的name属性的值，并将这个值注入到person的name属性中。因此，person的name属性的值将会是"New York"。

--- 
## **注解自定义赋值**：Spring 的许多注解支持 SpEL，如 `@Value`、`@PreAuthorize`、`@PostAuthorize`、`@Query` 

  **@Value：这个注解通常用于注入配置文件中的值。例如，我们可以使用SpEL来注入系统属性或环境变量的值：**
  ```java
  @Value("#{systemProperties['java.home']}")
  private String javaHome;
```
#{systemProperties['java.home']}是一个SpEL表达式，它获取了系统属性java.home的值，并将这个值注入到javaHome字段中。


**@PreAuthorize 和 @PostAuthorize：这两个注解用于方法级别的安全性检查。使用SpEL来显式表达安全性约束。例如，使用SpEL来检查当前用户是否有权限访问某个方法：**
```java
@PreAuthorize("hasRole('ADMIN')")
public void adminOnlyMethod() {
    // ...
}
```
hasRole('ADMIN')是一个SpEL表达式，它检查当前用户是否具有'ADMIN'角色。只有具有'ADMIN'角色的用户才能访问adminOnlyMethod方法。

---
## **数据验证**：使用SpEL和注解进行数据验证

  Spring 的数据验证 API 也支持 SpEL，可以用于复杂的数据验证场景，以及为测试类传入外部参数

首先，创建一个自定义的验证注解。在注解中使用SpEL表达式来定义验证规则：

```java
package io.github.maidsg.starter.start.annotation;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SpELAssertValidator.class)
@Documented
public @interface SpELAssert {

    String message() default "{SpELAssert.message}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    String value();
}
```

然后实现ConstraintValidator接口来创建一个自定义的验证器。在这个验证器中，我们将解析SpEL表达式，并使用这个表达式来验证字段的值
```java
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class SpELAssertValidator implements ConstraintValidator<SpELAssert, Object> {

    private ExpressionParser parser = new SpelExpressionParser();
    private String expression;

    @Override
    public void initialize(SpELAssert constraintAnnotation) {
        expression = constraintAnnotation.value();
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        return parser.parseExpression(expression).getValue(value, Boolean.class);
    }
}
```
在需要验证的字段上使用@SpELAssert注解：
在这个例子中，@SpELAssert(value = "email.contains('@')", message = "Email should contain @ symbol")是一个SpEL表达式，它检查email字段是否包含'@'字符。如果email字段不包含'@'字符，那么验证将失败，并返回指定的错误消息。
```java
public class User {

    @SpELAssert(value = "email.contains('@')", message = "Email should contain @ symbol")
    private String email;

    // ...
}
```

验证和使用方法可以在Spring Boot应用的启动类中添加一个CommandLineRunner bean，该bean在应用启动时运行，并创建并验证一个User对象
```java
package io.github.maidsg.starter.start.util;

import io.github.maidsg.starter.start.model.cases.User;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import java.util.Set;

@Component
public class ApplicationRunner implements CommandLineRunner {

    private Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

    @Override
    public void run(String... args) throws Exception {
        User user = User.builder().email("test.com").build();
        Set<ConstraintViolation<User>> violations = validator.validate(user);
        if (violations.isEmpty()) {
            System.out.println("Email is valid");
        } else {
            System.out.println("Email is invalid");
        }

        user = User.builder().email("test").build();
        violations = validator.validate(user);
        if (violations.isEmpty()) {
            System.out.println("Email is valid");
        } else {
            System.out.println("Email is invalid");
        }
    }
}
```

---
## **控制Bean注册**：配合@Conditional注解，控制Bean的注册
  使用SpEL（Spring Expression Language）在@Conditional注解中动态地注册bean仅当设置了某个特定的系统属性时才注册bean，达到@ConditionalOnProperty的效果。

先创建一个自定义的条件类，在这个类中，通过解析SpEL表达式，并使用这个表达式来决定是否注册bean：
```java
package io.github.maidsg.starter.start.condition;

import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.util.MultiValueMap;

public class OnExpressionCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(ConditionalOnExpression.class.getName());
        if (attributes != null) {
            for (Object value : attributes.get("value")) {
                String expression = (String) value;
                Boolean result = context.getEnvironment().getProperty(expression, Boolean.class, false);
                if (result) {
                    return true;
                }
            }
        }
        return false;
    }
}
```
创建一个自定义的@Conditional注解。在这个注解中指定上面创建的条件类，用于在方法中显式声明。
```java
package io.github.maidsg.starter.start.annotation;

import io.github.maidsg.starter.start.condition.OnExpressionCondition;
import org.springframework.context.annotation.Conditional;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(OnExpressionCondition.class)
public @interface ConditionalOnExpression {

    String value();
}
```
使用时，在需要创建Bean对象的方法上，使用注解并且传入指定的属性值，即可让spring检查条件，环境属性custom.bean.enabled的值。只有当这个属性的值为true时，customBean才会被注册。
```java
@Configuration
public class AppConfig {

    @Bean
    @ConditionalOnExpression("custom.bean.enabled")
    public CustomBean customBean() {
        return new CustomBean();
    }
}
```

---

## **Spring批处理**：在Spring批处理中动态配置步骤的输入和输出资源

使用SpEL（Spring Expression Language）来动态地配置作业参数、决定下一个要执行的步骤，或者配置步骤的输入和输出资源。

创建batch作业后，使用SpEL来动态地配置作业参数：
```java
@Configuration
@EnableBatchProcessing
public class BatchConfiguration {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Value("#{jobParameters['input.file.name']}")
    private Resource inputFile;

    @Bean
    public Job importUserJob(Step step1) {
        return jobBuilderFactory.get("importUserJob")
                .incrementer(new RunIdIncrementer())
                .flow(step1)
                .end()
                .build();
    }

    @Bean
    public Step step1(ItemReader<User> reader, ItemProcessor<User, User> processor, ItemWriter<User> writer) {
        return stepBuilderFactory.get("step1")
                .<User, User> chunk(10)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .next("#{jobExecutionContext['nextStep']}")
                .build();
    }

    @Bean
    public FlatFileItemReader<User> reader() {
        FlatFileItemReader<User> reader = new FlatFileItemReader<>();
        reader.setResource(inputFile);
        // configure reader...
        return reader;
    }

    // ...
}
```

案例中，通过使用@Value("#{jobParameters['input.file.name']}")注解来从作业参数中获取输入文件的名称后，就可以在reader()方法中使用这个文件名来创建一个FlatFileItemReader对象；
使用#{jobExecutionContext['nextStep']}表达式来从作业执行上下文中获取下一个要执行的步骤的名称，SpEL表达式会从作业执行上下文中获取下一个要执行的步骤的名称。如果作业执行上下文中没有nextStep属性，那么这个步骤将不会执行任何后续步骤。