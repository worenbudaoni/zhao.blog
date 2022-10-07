# 报错记录—feign转对象为linkedHashMap

> 为什么报错，给大家说个流程，有A、B两个微服务，B作为消费者通过 feign 调用 A 的接口。
>
> 我的想法是把，A 微服务中对象序列化之后存入 redis 做缓存，B 微服务作为 feign 的接收方，只需要在 redis 中将 key 对应的 value 反序列化存入对象中就行。
>
> 这里就出现了一个问题，B微服务 程序一直报错，卡在了 redis 获取数据的代码上，之后我在A、B两个微服务上各写入 redis 的 set 操作，发现两个序列化后的数据，竟然不一样，按道理说通过 RPC 调用后的 A微服务 中返回的数据应该和 B微服务 接收的参数是一样的，我进一步打印了 A、B微服务中 的返回值与接收值，发现竟然真的不一样，他把对象直接转成了Map集合

![image-20220929101106465](C:\Users\18358\AppData\Roaming\Typora\typora-user-images\image-20220929101106465.png)

# 原因

因为rpc远程调用在底层还是使用的HTTPClient，所以在传递参数的时候，必定要有个顺序，当你传递map的时候map里面的值也要有顺序，不然服务层在接的时候就出问题了，所以它才会从map转为linkedhashMap！spring 有一个类叫ModelMap，继承了linkedhashMap public class ModelMap extends LinkedHashMap ,所以一个接口返回的结果就可以直接用ModelMap来接，注意ModelMap是没有泛型的，不管你返回的结果是什么类型的map，泛型是多复杂的map，都可以直接new一个Modelmap，用它来接返回的结果！！！

# 解决方法

## 一、我的（   可能不太好，但至少能跑了，(σﾟ∀ﾟ)σ..:*☆     ）：

使用fastjson把List<对象>，转成字符串，再通过feign调用后，转为对应的对象，上代码

A服务

```java
    @Override
    public String getStudentReport(String studentCode) {
        try {
            //这些不用管
            StudentReportIndex reportIndexByStudentCode = getStudentReportIndexByStudentCode(studentCode);
            List<StudentReportVO> result = new ArrayList();
            if( null == reportIndexByStudentCode){
                return JSON.toJSONString(result);
            }
            for(ReportPageEnum pageEnum : ReportPageEnum.values()){
                PropertyCheck obj = pageEnum.getPageConvert().convert2VO(reportIndexByStudentCode,pageEnum.getVoClazz());
                if( (null == obj || obj.checkPropertyAllEmpty()) && pageEnum.equals(ACADEMIC_DATA)){
                    continue;
                }
                StudentReportVO reportVO = new StudentReportVO(pageEnum);
                reportVO.setT(obj);
                result.add(reportVO);
            }
            //主要的是这里的返回值
            return JSON.toJSONString(result);
        }catch (Exception e){
            e.printStackTrace();
            return e.getMessage();
        }
    }
```

B服务

feign调用

```java
    @GetMapping(value = "/接口")
    String getStudentReport(@RequestParam("studentCode") String studentCode);
```

实现类（主要部分）

```java
					// 查询redis
                    List<StudentReportVO> studentReportVOList = new ArrayList<>();
                    studentReportVOList = (List<StudentReportVO>) redisUtils.get(redisKey);
                    if (!CollectionUtils.isEmpty(studentReportVOList)) {
                        return R.Success("获取学生报告成功!", studentReportVOList);
                    }

                    // redis如果没有没有查询到就调用接口生成学生报告

					//通过fastjson来反编译成对象
                    ArrayList<StudentReportVO> arrayList = JSONObject.parseObject(studentReportService.getStudentReport(studentCode), ArrayList.class);
                    //Json封装类 R 装一下，因为我是做次开发，尽量不想动以前的代码
                    studentReport = R.Success(arrayList);
                    if (!CollectionUtils.isEmpty(studentReport.getData())) {
                        //存入redis中做缓存
                        redisUtils.set(redisKey, studentReport.getData(), expireTime, TimeUnit.DAYS);
                    }
```

## 二、通过序列化-反序列化获取（没试过）

具体没试过 ， 可以看这篇博客

[ Fegin调用的时候数据格式转换为LinkedHashMap的问题_llllllllll4er5ty的博客-CSDN博客_feign linkedhashmap](https://blog.csdn.net/llllllllll4er5ty/article/details/120065020)

## 三、dozer解决（没试过）

### 1.问题

spring cloud项目开发中，使用远程调用 ，返回类型为DataResults<List>，对List进行二次封装报java.lang.ClassCastException: java.util.LinkedHashMap cannot be cast to Users。

![img](https://img-blog.csdnimg.cn/9a6cad950ccb486581d83f91cce04831.png)

### 2. 解决方案：

写一个通用转换类

maven依赖

```xml
<dependency>
    <groupId>net.sf.dozer</groupId>
    <artifactId>dozer</artifactId>
    <version>5.5.1</version>
</dependency>
```

工具类：

```java
package com.bruce.utils;

import org.dozer.DozerBeanMapper;

import java.util.ArrayList;
import java.util.List;

public class GeneralConv {
    /**
     * 对象通用转换
     *
     * @param source           源对象
     * @param destinationClass 目标类
     * @param <T>
     * @return 返回得到destinationClass
     */
    public static <T> T conv(Object source, Class<T> destinationClass) {
        if(null == source){
            return null;
        }
        DozerBeanMapper dozerMapper = new DozerBeanMapper();
        T convObj = dozerMapper.map(source, destinationClass);
        return convObj;
    }

    /**
     * 集合转换
     *
     * @param sourceList       源集合
     * @param destinationClass 目标类
     * @param <T>
     * @return 返回得到destinationClass的集合结果
     */
    public static <T> List<T> convert2List(List<?> sourceList, Class<T> destinationClass) {
        List<T> destinationList =new ArrayList<>();
        sourceList.forEach(source -> {
            destinationList.add(GeneralConv.conv(source, destinationClass));
        });
        return destinationList;
    }
}
```

### 3.使用

如果需要使用对象中的数据，可以自己调用工具类转换

![img](https://img2022.cnblogs.com/blog/2803381/202203/2803381-20220322204038567-221355689.png)



此解决方案来自[SpringCloud对象接收时候对象变成LinkeHashMap解决方案 - cr-forever - 博客园 (cnblogs.com)](https://www.cnblogs.com/code-cr/p/16041248.html#gallery-2)