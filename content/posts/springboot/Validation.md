---
title: "Validation"
date: 2022-12-04T23:35:31+08:00
draft: false
categories: ["springboot","validation"]
---

# 背景

Java API规范(JSR303)定义了Bean校验的标准validation-api，但没有提供实现。hibernate validation是对这个规范的实现，并增加了校验注解如@Email、@Length等。

Spring Validation是对hibernate validation的二次封装，用于支持spring mvc参数自动校验。接下来，我们以spring-boot项目为例，介绍Spring Validation的使用。

**主要是对http 请求传递过来的参数进行校验，提前暴露问题**

# 使用

如果spring-boot版本小于2.3.x，spring-boot-starter-web会自动传入hibernate-validator依赖。如果spring-boot版本大于2.3.x，则需要手动引入校验组件依赖：

```xml
        <!--校验组件-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
```

**注：如果Spring-boot 版本小于2.3.x ,但是手动引入hibernate-validator 版本较大的时候，可能会出现校验无效的情况**，

**在Controller层一定要做参数校验的**！大部分情况下，请求参数分为如下两种形式：

- POST、PUT请求，使用requestBody传递参数；
- GET请求，使用`requestParam/PathVariable`传递参数。

## 常用校验注解

原生javax.validation 里面支持的注解

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212060032442.png" alt="image-20221206003244303" style="zoom:50%;" />

hibernate-validator  里面支持的校验注解

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212060035686.png" alt="image-20221206003537657" style="zoom:50%;" />

## 校验Body参数

Body参数一般都是个对象，我们可以在对象中通过注解来校验对应的字段

1. 在对象上添加@Validated 注解

2. 在对应的属性上根据不同的规则设置不同的注解

   ```java
   /**
    * 员工对象
    *
    * @author peilizhi
    * @date 2022/12/6 00:15
    **/
   @Data
   public class UserDO {
   
       /**
        * 最大长度32
        */
       @Length(max = 32, message = "userId最大长度为32位")
       private String userId;
       /**
        * 不为空
        */
       @NotNull(message = "name 不能为空")
       private String name;
       
       @Min(value = 0, message = "年龄最小值为0")
       private Integer old;
       
       /**
        * 自定义固定值校验
        */
       @FixedValueValidator(fixedValue = {"boy", "girl"}, message = "性别有误")
       private String sex;
   
       @Length(min = 11, max = 11, message = "手机号只能为11位")
       @Pattern(regexp = "^[1][3,4,5,6,7,8,9][0-9]{9}$", message = "手机号格式有误")
       private String phone;
   
       @Email(message = "邮箱格式不正确")
       private String email;
   }
   
   ```

   自定义校验类

   ```java
   /**
    * 固定值校验
    *
    * @author by peilizhi
    * @date 2022/12/6 00:46
    */
   @Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
   @Retention(RUNTIME)
   @Documented
   @Constraint(validatedBy = {FixedValueValidator.FixedValueValid.class})
   public @interface FixedValueValidator {
       String message() default "FixedValue's value is invalid";
   
       Class<?>[] groups() default {};
   
       Class<? extends Payload>[] payload() default {};
   
       String[] fixedValue();
   
       class FixedValueValid implements ConstraintValidator<FixedValueValidator, Object> {
   
           String[] fixedValue = null;
   
           @Override
           public void initialize(FixedValueValidator validData) {
               this.fixedValue = validData.fixedValue();
           }
   
           /**
            * 校验值是否在固定值范围里面
            *
            * @param value 待校验的值
            */
           @Override
           public boolean isValid(Object value, ConstraintValidatorContext constraintContext) {
               if (fixedValue == null || fixedValue.length == 0) {
                   return false;
               }
               if (value == null) {
                   return true;
               }
               boolean flag = false;
               for (String str : fixedValue) {
                   if (String.valueOf(value).equals(String.valueOf(str))) {
                       flag = true;
                       break;
                   }
               }
               return flag;
           }
       }
   }
   ```

3. 接口参数上标注校验

   ```java
   @PostMapping("insert-user")
       public String insertUser( @Validated UserDO userDO) {
           userDO.setUserId(UUID.randomUUID().toString());
           log.info("user:{}", JSONUtil.toJsonStr(userDO));
           return userDO.getUserId();
       }
   ```

   

4. 处理校验失败的情况

   + 校验失败的时候会抛出 MethodArgumentNotValidException 异常，http 请求返回400（Bad request）
   + 为了更友好的展示校验错误，需要对MethodArgumentNotValidException 异常进行处理

![image-20221206013520329](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212060135412.png)

```java
/**
 * 异常捕获处理类
 *
 * @author peilizhi
 * @date 2022/12/6 01:37
 **/
@RestControllerAdvice
public class CommonExceptionHandler {

    @ExceptionHandler(value = {BindException.class})
    @ResponseBody
    public ModelAndView handleBindException(BindException ex) {
        BindingResult bindingResult = ex.getBindingResult();
        StringBuilder sb = new StringBuilder("校验失败:");
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            sb.append(fieldError.getField()).append("：").append(fieldError.getDefaultMessage()).append(", ");
        }
        String msg = sb.toString();

        MappingJackson2JsonView view = new MappingJackson2JsonView();
        Map<String, Object> model = new HashMap<>(8);
        model.put("code", 500);
        model.put("message", msg);
        model.put("status", false);
        view.setAttributesMap(model);


        ModelAndView mav = new ModelAndView();
        mav.setView(view);
        return mav;
    }

    @ExceptionHandler(value = {MethodArgumentNotValidException.class})
    @ResponseBody
    public ModelAndView handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
        BindingResult bindingResult = ex.getBindingResult();
        StringBuilder sb = new StringBuilder("校验失败:");
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            sb.append(fieldError.getField()).append("：").append(fieldError.getDefaultMessage()).append(", ");
        }
        String msg = sb.toString();

        MappingJackson2JsonView view = new MappingJackson2JsonView();
        Map<String, Object> model = new HashMap<>(8);
        model.put("code", 500);
        model.put("message", msg);
        model.put("status", false);
        view.setAttributesMap(model);


        ModelAndView mav = new ModelAndView();
        mav.setView(view);
        return mav;
    }


    @ExceptionHandler({ConstraintViolationException.class})
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public ModelAndView handleConstraintViolationException(ConstraintViolationException ex) {
        MappingJackson2JsonView view = new MappingJackson2JsonView();
        Map<String, Object> model = new HashMap<>(8);
        model.put("code", 500);
        model.put("message", ex.getMessage());
        model.put("status", false);
        view.setAttributesMap(model);


        ModelAndView mav = new ModelAndView();
        mav.setView(view);
        return mav;
    }
}

```

**注： 在spring-boot 2.3.x 之前返回的异常类是BindException 之后才是ConstraintViolationException、MethodArgumentNotValidException 异常**

## 校验路径

**这种在参数中增加校验注解的话，必须在类上标注@Validated 注解，如果不加的话，注解不起作用**

```java
/**
 * @author peilizhi
 * @date 2022/12/6 00:54
 **/
@Slf4j
@RestController
@RequestMapping("validate")
@Validated
public class UserController { 
@GetMapping("{userId}")
    public String queryUser(@PathVariable("userId") @FixedValueValidator(fixedValue = {"11", "22"})String userId){
        return userId;
 }
}
```

## 校验参数

**这种在参数中增加校验注解的话，必须在类上标注@Validated 注解，如果不加的话，注解不起作用**

```java
/**
 * @author peilizhi
 * @date 2022/12/6 00:54
 **/
@Slf4j
@RestController
@RequestMapping("validate")
@Validated
public class UserController {
@GetMapping(value = "get-user")
    public String getUser(@Length(min = 1,max = 12) String userId) {
        final String uuid = UUID.randomUUID().toString();
        log.info("user:{}", JSONUtil.toJsonStr(uuid));
        return uuid;
    }
}

```

## 分组校验

有的情况下，字段是必须有值的，但是在另一种情况下是不需要有值的，这个时候就不能使用一个注解来处理这种复杂的情节，需要使用分组注解实现。

需要先定义接口，用来表明分组,接口没有特殊函数，只是声明分组

```java
/**
 * @author peilizhi
 * @date 2022/12/8 00:33
 **/
public interface Update {
}


/**
 * @author peilizhi
 * @date 2022/12/8 00:33
 **/
public interface Insert {
}
```



```java
/**
 * 员工对象
 *
 * @author peilizhi
 * @date 2022/12/6 00:15
 **/
@Data
public class UserDO {

    /**
     * 最大长度32
     */
    @Length(max = 32, message = "userId最大长度为32位")
    @NotNull(groups = {Update.class})
    private String userId;
    /**
     * 不为空
     */
    @NotNull(message = "name 不能为空", groups = {Update.class, Insert.class})
    private String name;

    @Min(value = 0, message = "年龄最小值为0", groups = {Update.class, Insert.class})
    private Integer old;
    /**
     * 自定义固定值校验
     */
    @FixedValueValidator(fixedValue = {"boy", "girl"}, message = "性别有误", groups = {Update.class, Insert.class})
    private String sex;

    @Length(min = 11, max = 11, message = "手机号只能为11位")
    @Pattern(regexp = "^[1][3,4,5,6,7,8,9][0-9]{9}$", message = "手机号格式有误")
    private String phone;

    @Email(message = "邮箱格式不正确")
    private String email;
}
```

在使用的时候需要在@Validated 里面表明使用的哪个分组

```java
@PostMapping(value = "insert-user")
    public String insertUser(@RequestBody @Validated(Insert.class) UserDO userDO) {
        userDO.setUserId(UUID.randomUUID().toString());
        log.info("user:{}", JSONUtil.toJsonStr(userDO));
        return userDO.getUserId();
    }

    @PostMapping(value = "update-user")
    public String updateUser(@RequestBody @Validated(Update.class) UserDO userDO) {
        userDO.setUserId(UUID.randomUUID().toString());
        log.info("user:{}", JSONUtil.toJsonStr(userDO));
        return userDO.getUserId();
    }
```

***注：明确指定@Validated里面使用哪个分组的话，没有配置分组的注解就不生效***

## 嵌套校验

有的时候属性可能不光简单是String ，也有可能是对象形式的话，也是可以通过@Valid 校验对象里面的属性.并且@Valid不能缺少

在这里可以嵌套集合或者map。集合对象的话就是校验集合中的每一个对象，map的话就是校验map的value

```java
/**
 * 员工对象
 *
 * @author peilizhi
 * @date 2022/12/6 00:15
 **/
@Data
public class UserDO {

    /**
     * 最大长度32
     */
    @Length(max = 32, message = "userId最大长度为32位")
    @NotNull(groups = {Update.class})
    private String userId;
    /**
     * 不为空
     */
    @NotNull(message = "name 不能为空", groups = {Update.class, Insert.class})
    private String name;

    @Min(value = 0, message = "年龄最小值为0", groups = {Update.class, Insert.class})
    @Max(value = 150, message = "年龄最大150岁")
    private Integer old;
    /**
     * 自定义固定值校验
     */
    @FixedValueValidator(fixedValue = {"boy", "girl"}, message = "性别有误", groups = {Update.class, Insert.class})
    private String sex;

    @Length(min = 11, max = 11, message = "手机号只能为11位")
    @Pattern(regexp = "^[1][3,4,5,6,7,8,9][0-9]{9}$", message = "手机号格式有误")
    @NotNull
    private String phone;

    @Email(message = "邮箱格式不正确")
    private String email;

    @NotNull(message = "地址不能为空", groups = {Insert.class, Update.class})
    @Valid
    private List<Address> address;
    
    @NotNull(message = "地址不能为空", groups = {Insert.class, Update.class})
    @Valid
    private Map<String,Address> addressMap;
}
```

## 集合校验

如果参数直接是个集合的话，我们也想要校验集合里面的每一对象的话。直接使用`java.util.Collection`下的list或者set来接收数据，参数校验并不会生效！我们可以使用自定义list集合来接收参数。这种集合校验就是参考嵌套校验

```java
/**
 * @author peilizhi
 * @date 2022/12/8 01:04
 **/
public class ValidationList<E>{

    /**
     * 一定要加@Valid注解
     */
    @Valid
    @Delegate
    public List<E> list = new ArrayList<>();
}
```

@Delegate是 lombook 的注解，用于为对象生成一些方法。可以理解为A、B 有相同的方法，但是调用A 里面方法的时候实际是调用B里面的方法

```java
@PostMapping(value = "batch-insert-user")
    public String batchInsertUser(@RequestBody ValidationList<UserDO> UserList) {
        UserList.forEach(userDO -> {
            userDO.setUserId(UUID.randomUUID().toString());
            log.info("user:{}", JSONUtil.toJsonStr(userDO));
        });
        return "OK";
    }
```

## 自定义校验

1. 先定义校验注解（指标注解作用域，作用时机）
2. 添加 @Constraint  指明通过哪个类来校验
3. 实现校验类
4. 校验类需要实现ConstraintValidator <自定义注解，待校验的对象类型>注解

```java
/**
 * 固定值校验
 *
 * @author by peilizhi
 * @date 2022/12/6 00:46
 */
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {FixedValueValidator.FixedValueValid.class})
public @interface FixedValueValidator {
    String message() default "FixedValue's value is invalid";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    String[] fixedValue();

    class FixedValueValid implements ConstraintValidator<FixedValueValidator, Object> {

        String[] fixedValue = null;

        @Override
        public void initialize(FixedValueValidator validData) {
            this.fixedValue = validData.fixedValue();
        }

        /**
         * 校验值是否在固定值范围里面
         *
         * @param value 待校验的值
         */
        @Override
        public boolean isValid(Object value, ConstraintValidatorContext constraintContext) {
            if (fixedValue == null || fixedValue.length == 0) {
                return false;
            }
            if (value == null) {
                return true;
            }
            boolean flag = false;
            for (String str : fixedValue) {
                if (String.valueOf(value).equals(String.valueOf(str))) {
                    flag = true;
                    break;
                }
            }
            return flag;
        }
    }
}
```

## 编程式校验

编程式校验意味着不是通过注解自动校验，而是通过手动编码来校验

```java
@Autowired
    private javax.validation.Validator globalValidator;

    @PostMapping(value = "manual-insert-user")
    public String manualInsertUser(@RequestBody UserDO userDO) {
        log.info("user0:{}", JSONUtil.toJsonStr(userDO));
        // 手动校验
        final Set<ConstraintViolation<UserDO>> validate = globalValidator.validate(userDO);
        if (!validate.isEmpty()) {
            // 校验失败
            return "ERROR";
        }
        userDO.setUserId(UUID.randomUUID().toString());

        log.info("user:{}", JSONUtil.toJsonStr(userDO));
        return userDO.getUserId();
    }
```

## 快速失败

校验是全部校验完成之后才结束，可以配置只有一个错误就提前退出

```java
```

