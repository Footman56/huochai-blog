---
title: "Js计算引擎"
date: 2022-11-29T16:30:31+08:00
draft: false


categories: ["java","Nashorn","JavaScript"]
---

# 背景

最近工作上有需求需要了解 java 整合 js 计算引擎，就看了看同事之前写的代码，写的真好，对我有一种醍醐灌顶的感觉，也借此机会学习学习js 计算引擎

项目中使用的是 Nashorn 引擎，

[Nashorn官网](https://docs.oracle.com/javase/10/nashorn/introduction.htm#JSNUG136)



# 使用

Nashorn 引擎计算的本质，在执行一个j s 函数，函数中可以自定义一些变量和函数，也可以使用j s 原生的函数，在使用自定义函数的时候需要告诉函数是怎么运行的。计算的结果也放在这个函数的作用域中，可以有多个计算结果

```java
import javax.script.*;

public class EvalScript {
    public static void main(String[] args) throws Exception {
        // create a script engine manager
        ScriptEngineManager factory = new ScriptEngineManager();
        // create a Nashorn script engine
        ScriptEngine engine = factory.getEngineByName("nashorn");
        // evaluate JavaScript statement
        try {
          // 这里执行的时候 等同于执行 js脚本 
            engine.eval("print('Hello, World!');");
        } catch (final ScriptException se) { se.printStackTrace(); }
    }
}
```

 如果要使用自定义函数、变量的话，就需要再引擎里面声明这些自定义函数、变量。

```java
/**
     * 公式计算
     *
     * @param params      参数
     */
    public String calcuate(List<String> formulationList, Map<String, Double> params) throws Exception {
        if (CollectionUtils.isEmpty(formulationList)) {
            return StringUtils.EMPTY;
        }
        // 获取自定义解析器
        ScriptEngine engine = factory.getScriptEngine();
        for (IScriptStrategy iScriptStrategy : factory.getScriptStrategyList()) {
            final EFunctionEnum func = iScriptStrategy.getFunc();
            // 绑定解析器
            String key = func.toString();
            engine.put(key, iScriptStrategy);
            // 绑定到引擎上,用于绑定自定义函数
            engine.eval("Object.bindProperties(this, " + key + ")");
        }
        // 绑定参数
        params.forEach(engine::put);

        // 计算
        for (String formulation : formulationList) {
            engine.eval(formulation);
        }
        // 可以获取多个返回,直接从引擎到作用域中获取
        final Object vale = engine.get("得分");
        final Object o = engine.get("判断");
        final String doubleValue = DoubleUtil.handleDoubleValue(o);
        System.out.println("判断 = " + doubleValue);
        System.out.println("DoubleUtil.handleDoubleValue(vale) = " + DoubleUtil.handleDoubleValue(vale));
        return DoubleUtil.handleDoubleValue(vale);
    }
```

## put

put方法就是向解析器注册一个变量、函数

![image-20221130165429843](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202211301654377.png)

## Object.bindProperties

这个方法用于向解析器绑定自定义的函数。需要先定义类，之后在将类中的所有方法放到解析器中，比较耗时

**这里绑定的函数 需要和公式里面的函数名称保持一致**

![image-20221130165634288](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202211301656324.png)

## get

从解析器中获取对象，常用于换取结果。j s 中可以定义多个赋值语句，get 就能获取赋值语句的结果

![image-20221130165835375](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202211301658416.png)



**注：这里计算的时候很需要关注字段类型。**

字符串 与 Double 计算逻辑不一致的。

举例： + 运算符对应 字符串来说就是拼接，对于Double 就是相加运算

再举例： > 运算符对于 字段串来说 “5”> "10" true