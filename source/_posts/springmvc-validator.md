---
title: springmvc-validator
comments: true
date: 2019-04-23 22:04:55
tags: 
    - springmvc
    - validator
---

# Spring Boot 数据校验框架 springmvc-validator 使用说明


## 使用步骤

> **依赖**

`hibernate-validation`,`validation-api`已经被添加在`spring-boot-starter-web`内，所以不需要添加依赖

<!--more-->

> **配置**

SpringMVC的`javabean` 配置方式如下所示
> 注意：快速失败方式才需要配置下面的，默认可以不用配置。


```java

@Bean
public Validator validator(){
    ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
            .configure()
            .addProperty( "hibernate.validator.fail_fast", "true" ) //为true时代表快速失败模式，false则为全部校验后再结束。
            .buildValidatorFactory();
    Validator validator = validatorFactory.getValidator();

    return validator;
}


```


## 使用
### 两种常规校验
> 1. 在`Controller`类上加入`@Validated`注解
> 2. 在方法参数中添加相应的注解
> 3. 注意：若方法参数上没有`@Validated`注解，则需要在类上加`@Validated`注解

> 参数为实体类对象
```java
@Data
public class SysRole {
    
    private String id;
    
    @NotNull(message = "角色名称不能为空")
    private String name;
    
    @Size(min = 6, max = 20, message = "描述必须在6-20个字符之间")
    private String description;

    @Column(name = "create_time")
    private Date createTime;
}


@RequestMapping("/demo")
public void demo(@Valid SysRole sysRole, BindingResult result){
	if (result.hasErrors()) {
	     List<ObjectError> allErrors = result.getAllErrors();
	     StringBuilder sb = new StringBuilder();
	     for (ObjectError objectError : allErrors) {
	         sb.append(objectError.getDefaultMessage() + ";");
	     }
	     throw new ServiceException(sb.toString());  // 此处一定要抛出异常，否则校验框架不起作用。
	 }
    //do something  
}

```
> 参数为单个字段
```java
@GetMapping("/demo")
public void demo2(@Min(value = 1, message = "班级最小只能1")
                   @Max(value = 99, message = "班级最大只能99")
                   @RequestParam(name = "classroom", required = true) int classroom) {
                 //do something  
}  

```
### 分组校验
> **1.新建两个接口**
```java
public interface RoleUpdate {
}

public interface RoleAdd {
}
```

> **2.实体类中给字段加入分组**
```java
@Data
public class SysRole {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @NotEmpty(message = "角色ID不能为空", groups = {RoleUpdate.class})
    protected String id;

    @NotEmpty(message = "角色名称不能为空", groups = {RoleAdd.class})
    private String name;

    private String description;
}
```

> **3.Controller中的使用**
```java
@RestController
@RequestMapping("/sys/role")
public class SysRoleController {

    @PostMapping("/add")
    public RestResult add(@Validated({RoleAdd.class}) SysRole sysRole, BindingResult result){
        CoreUtil.validationHandler(result);
        sysRoleService.addRole(sysRole);
        return ResultGenerator.genSuccessResult();
    }

    @PostMapping("/update")
    public RestResult update(@Validated({RoleUpdate.class}) SysRole sysRole, BindingResult result) {
        CoreUtil.validationHandler(result);
        sysRoleService.update(sysRole);
        return ResultGenerator.genSuccessResult();
    }
}
```
> **4.组序列的使用：**

```java
//指定组的验证顺序，前面组验证不通过的，后面组不进行验证
@GroupSequence({RoleAdd.class, RoleUpdate.class, Default.class})
public interface GroupOrder {
}
```

             
    
##  常见的注解：
> 所有注解均有message这个参数，即你需要返回或捕获的内容
- `@Null(message="")`   被注释的元素必须为 null     
- `@NotNull`   被注释的元素必须不为 null     
- `@AssertTrue`     被注释的元素必须为 true     
- `@AssertFalse`    被注释的元素必须为 false     
- `@Min(value)`     被注释的元素必须是一个数字，其值必须大于等于指定的最小值     
- `@Max(value)`    被注释的元素必须是一个数字，其值必须小于等于指定的最大值     
- `@DecimalMin(value)`  被注释的元素必须是一个数字，其值必须大于等于指定的最小值     
- `@DecimalMax(value)`  被注释的元素必须是一个数字，其值必须小于等于指定的最大值     
- `@Size(max=, min=)`   被注释的元素的大小必须在指定的范围内     
- `@Digits (integer, fraction)`     被注释的元素必须是一个数字，其值必须在可接受的范围内     
- `@Past`   被注释的元素必须是一个过去的日期     
- `@Future`    被注释的元素必须是一个将来的日期     
- `@Pattern(regex=,flag=)`  被注释的元素必须符合指定的正则表达式     
- `@Email`  被注释的元素必须是电子邮箱地址     
- `@Length(min=,max=)`  被注释的字符串的大小必须在指定的范围内     
- `@NotEmpty`   被注释的字符串的必须非空     
- `@Range(min=,max=,message=)`  被注释的元素必须在合适的范围内
    
## 异常信息处理

```java

if(exception instanceof ConstraintViolationException){
    ConstraintViolationException exs = (ConstraintViolationException) exception;
    Set<ConstraintViolation<?>> violations = exs.getConstraintViolations();
    for (ConstraintViolation<?> item : violations) {
        //打印验证不通过的信息
        System.out.println(item.getMessage());
    }
}

```