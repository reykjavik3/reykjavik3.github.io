---
layout: post
title: aop+spel表达式在一个通用的操作记录模块中的实践
categories: business
description: aop+spel表达式在一个通用的操作记录模块中的实践
keywords: java
---
![](https://user-gold-cdn.xitu.io/2020/6/27/172f594c69f27290?w=658&h=931&f=png&s=851588)
先祝大家端午快乐
# 先说背景

一般应用服务中都会有记录某个数据变化轨迹的需求，比如我们用户中心，会记录某个用户从注册到使用中更换过手机号，更换用户名的日志记录，再到最后注销等一系列操作过程中的数据轨迹，早期的记录是侵入到业务中的，每个服务有记录数据轨迹的需求，就单手撸一套数据轨迹的功能

![](https://user-gold-cdn.xitu.io/2020/6/27/172f5a2d6f4b30bc?w=550&h=351&f=png&s=454490)
这样做的缺点是侵入业务，而且每次都要开发一套，效率低，开发心里苦，眼泪完全忍不住；


那怎么办，能不能开发一套通用的数据轨迹模块，给大家复用起来，再有类似的需求，能不能不重复开发，一个注解就接入数据轨迹的功能，那开发们就该笑出声了；

![](https://user-gold-cdn.xitu.io/2020/6/27/172f5a4c54890a89?w=554&h=312&f=png&s=409534)

# 实现原理

第一个想法就是aop切面搞定，使用时一个注解放到方法上边，aop切对应注解，在aop中记录日志，对应的使用方只要告诉我操作的一些基本信息（数据类型，操作类型等），我就全部帮他搞定；
    但是问题来了

  1 在切面中我需要知道历史值是多少，那我如何在切面中实现一套通用的历史数据查询接口

  2 想查询历史数据，我就需要id,但是切面拿到的是可能是多个参数，而且这些参数中还是引用关系，比如一个对象是班级，班级里边有学生，那学生id是我要的数据，但是这个对象嵌套的层次是不固定的，怎么拿到这个id；

对于第一个问题，我们让使用者告诉我们操作的是哪个表，把表名配置在对应的注解里，提供一套通用的数据查询接口就ok了，直接上代码。
```java
/**
     * 查询任意sql
     * @param table：表名,id:id
     * @return Map
     */
    @Select("SELECT * FROM ${table} where id=#{id}")
    Map<String, Object> selectAnyTalbe(@Param("table") String table,@Param("id")String id);
```

对应操作方法的参数中，可能有好几个对象，比如这个方法
 ``` java
 public ResultObject<MPCUser> updateSchoolClassStudent(School school, Person person, User user) {
        return new ResultObject<>();
    }
 ```
 ```java
 public class School {
    private SchoolClass schoolClass;
}
public class SchoolClass {
    private Student student;
}
public class Student {
    private String id;
}
 ```
我想拿
```
school.getSchoolClass().getStudent().getId();
```
正常业务代码可以这样，但我们通用的aop切面不知道对象名字，乃至对象有多少层嵌套，那怎么办？用spel表达式


# spel表达式
 ##  用法
 spel表达式有三种用法

 1 @Value
```
    //@Value能修饰成员变量和方法形参
    //#{}内就是表达式的内容
    @Value("#{表达式}")
    public String arg;
```
2 spring \<bean\>配置
```
<bean id="xxx" class="com.java.XXXXX.xx">
    <!-- 同@Value,#{}内是表达式的值，可放在property或constructor-arg内 -->
    <property name="arg" value="#{表达式}">
</bean>
```
3 Expression
```java
public static void main(String[] args) {

        //创建ExpressionParser解析表达式
        ExpressionParser parser = new SpelExpressionParser();
        //表达式放置
        Expression exp = parser.parseExpression("表达式");
        //执行表达式，默认容器是spring本身的容器：ApplicationContext
        Object value = exp.getValue();

        /**如果使用其他的容器，则用下面的方法*/
        //创建一个虚拟的容器EvaluationContext
        StandardEvaluationContext ctx = new StandardEvaluationContext();
        //向容器内添加bean
        BeanA beanA = new BeanA();
        ctx.setVariable("bean_id", beanA);

        //setRootObject并非必须；一个EvaluationContext只能有一个RootObject，引用它的属性时，可以不加前缀
        ctx.setRootObject(XXX);

        //getValue有参数ctx，从新的容器中根据SpEL表达式获取所需的值
        Object value = exp.getValue(ctx);
    }
```
表达式的语法也很简单，支持直接赋值，引用赋值，运算符赋值，比较，逻辑，条件，正则等赋值方法；

我们这次用的第三种Expression
先看使用效果
``` java
@OperationLog(name="更新用户",table="student",type= OperationType.UPDATE,idKey = "#school.schoolClassStudent.id")
public ResultObject<MPCUser> updateSchoolClassStudent(School school, Person person, User user) {
       return new ResultObject<>();
   }
```
我在aop中就能拿到使用方配置的
```java
#school.schoolClassStudent.id
```
这个参数

问题又来了，怎么拿？上代码

```java
public String doKey(ProceedingJoinPoint joinPoint, OperationLog operationlog) {
        //获取方法的参数名和参数值
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        List<String> paramNameList = Arrays.asList(methodSignature.getParameterNames());
        List<Object> paramList = Arrays.asList(joinPoint.getArgs());

        //将方法的参数名和参数值一一对应的放入上下文中
        EvaluationContext ctx = new StandardEvaluationContext();
        for (int i = 0; i < paramNameList.size(); i++) {
            ctx.setVariable(paramNameList.get(i), paramList.get(i));
        }

        SpelExpressionParser spelExpressionParser = new SpelExpressionParser();
        // 解析SpEL表达式获取结果
        String value = spelExpressionParser.parseExpression(operationlog.idKey()).getValue(ctx).toString();
        return value;
    }
```
这个spel表达式获取到的value就是我们这条sql的id;
``` sql
 @Select("SELECT * FROM ${table} where id=#{id}")
```

看下我的OperationLog自定义注解
``` java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface OperationLog {
    /**
     * 业务名
     */
    String name();
    /**
     * 表名
     */
    String table();

    /**
     * 操作类型
     */
    OperationType type();

    /**
     * 操作主键的spel表达式
     */
    String idKey() default "";

    String operationReason() default "";

}
```

### 实现细节
数据库表
``` sql
CREATE TABLE `operate_hoistory` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '需要',
  `primary_key` varchar(64) DEFAULT NULL COMMENT '主键id',
  `table_name` varchar(64) DEFAULT NULL COMMENT '表名',
  `operate_type` varchar(8) DEFAULT NULL COMMENT '操作类型(add,update,delete);',
  `operate_time` timestamp NULL DEFAULT NULL COMMENT '操作时间',
  `org_data` text COMMENT '原先数据',
  `targ_data` text COMMENT '目标数据',
  `operate_reason` varchar(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `key_primarykey_tablename` (`primary_key`,`table_name`)
) ENGINE=InnoDB AUTO_INCREMENT=143571 DEFAULT CHARSET=utf8;
```
具体分三种情况  新增，删除，修改

新增时不用查询原始值，直接插入targ_data字段即可，入口即化

删除时查一次原始值，插入org_data，收工

![](https://user-gold-cdn.xitu.io/2020/6/27/172f5d0ede2bd874?w=557&h=481&f=png&s=245471)


重点是更新，反复对比更新的字段和原始值是否相同，如果不同(代表被更新了)，那么在org_data和targ_data中分别插入对应的字段值即可；

看下效果吧

我把用户的user_name修改了，对应的记录中会插入一条
![](https://user-gold-cdn.xitu.io/2020/6/27/172f5d439dce9978?w=1132&h=92&f=png&s=36028)

另外，持久层我使用的Mybatis，使用方是否开启驼峰命名我都支持；

使用方也很方便，在注解中配置好涉及的操作类型，表名，主键位置，就可以早点下班了。

## 效率问题
 能异步都异步，不用操心。

# 后记
后边脱敏了把代码传到github上

端午三天，杭州下了三天的雨，下午出去吃饭，天终于是晴了，在这雨后初晴的傍晚，太阳晒的人脊背发烫，像是冬天背靠着火炉一样，想着，走着，看看天上的云，想起了


> 那一天我二十一岁，在我一生的黄金时代，我有好多奢望。我想爱，想吃，还想在一瞬间变成天上半明半暗的云
