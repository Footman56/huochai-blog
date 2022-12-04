---
title: "PageHelper"
date: 2022-12-04T23:44:58+08:00
draft: false
categories: ["数据库"]
---

pageHelper是一种分页的插件。

```
pageHelper将分页数据写入到ThreadLocal中，在后面执行SQL请求的时候，再从ThreadLocal中提取出来，拼凑在SQL里面
```





# 一、引入jar

```xml
<pagehelper.version>1.2.7</pagehelper.version>

		<dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
    		<version>${pagehelper.version}</version>
        </dependency>
```

# 二、编写工具类

```java
/**
 * @Author peilizhi
 * @Date 2021/6/3 19:45
 **/
public class PageHelperUtil {

    /**
     * 分页查询
     *
     * @param param    分页参数
     * @param runnable 分页操作
     * @return 查询结果
     */
    public static <T> Page<T> doPage(PageParam param, Runnable runnable) {
        return doPage(param.getPageNum(), param.getPageSize(), runnable);
    }

    /**
     * 分页查询
     *
     * @param pageNum  分页当前页数，pageNum<=0为第一页
     * @param pageSize 每页条数
     * @param runnable 查询操作
     * @return 查询结果
     */
    public static <T> Page<T> doPage(Integer pageNum, Integer pageSize, Runnable runnable) {

        Page<T> page = PageHelper.startPage(pageNum, pageSize);
        try {
            runnable.run();
        } finally {
            PageHelper.clearPage();
        }
        return page;
    }

    /**
    *结构转换
    */
    public static <T, R> PageDO<R> convertPageModel(Page<T> page, Function<T, R> func) {
        PageDO<R> pageDo = new PageDO<>();
        pageDo.setPageNum(page.getPageNum());
        pageDo.setPageSize(page.getPageSize());
        pageDo.setTotalCount(page.getTotal());
        pageDo.setTotalPages(page.getPages());
        pageDo.setResult(page.getResult().stream().map(func).collect(Collectors.toList()));
        return pageDo;
    }
}
```

```JAVA
/**
 * @Author peilizhi
 * @Date 2021/6/3 19:46
 * 查询参数的父类，在子类中写查询的条件
 **/
@Data
public class PageParam {

    /**
     * 页数, 从 0 开始算第一页
     *
     * @required
     */
    private int pageNum = 0;

    /**
     * 每页大小
     *
     * @required
     */
    private int pageSize = 10;
}
```

```
先设置查询的页码
```

```java
public List<WindPowerEquipmentActivePO1> windPowerEquipmentActivePOList(Integer pageNo, Integer pageSize) {
        final Page<WindPowerEquipmentActivePO1> objectPage = PageHelperUtil.doPage(pageNo, pageSize, () -> mapper.selectAll());

        System.out.println("objectPage.getCountColumn() = " + objectPage.getCountColumn());
        List<WindPowerEquipmentActivePO1> result = objectPage.getResult();
        System.out.println("objectPage.getResult() = " + result);
        System.out.println("objectPage.getPages() = " + objectPage.getPages());
        return result;
    }
```

