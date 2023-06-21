---
title: mybatisPlus在注解上的问题
date: 2023-06-21 14:08:44
tags: 
 - spring
 - mybatis
---

### 场景

维护一个老项目，并在其中添加迭代一个新功能

发现依赖库中有mybatisPlus，于是就用了

### 依赖不全

> Invalid bound statement (not found)

检查发现是直接依赖的mybatis-plus-2.1.0gamma的版本

改为适配spring的版本，添加dependency

``` xml
<dependency>
   <groupId>com.baomidou</groupId>
   <artifactId>mybatisplus-spring-boot-starter</artifactId>
   <version>1.0.4</version>
</dependency>
```

### Mapper校验失败

引入starter后，原本正常的Mapper校验不通过了

> Dots are not allowed in element names，please remove it from

有issue说是interface和mapper.xml加载顺序的原因，查到的解决方案是mybatisPlus升级到3.1.x

[建议升级到3.1.x](https://blog.csdn.net/codeblf2/article/details/102687864)

[维护者的讨论](https://gitee.com/baomidou/mybatis-plus/issues/IYQI9)

### MybatisPlus升级到3.1.0

2.1升级到3.1，涉及到很多类的包路径变动，比如注解，Page等，改动量很大

但是start要升级到spring cloud 2.0，这对老项目来说不现实

放弃升级mybatis-plus

### 查源码

在Mapper.java中出现自定义的resultMapId时，必现

* 自定义resultMapId

``` java
@Select("<script> ... </script> ")
@Results(id = "appMap",
            value = {
                    @Result(property = "id", column = "id"),
                    @Result(property = "systemName", column = "system_name")
            })
List<PO> findSystemCallApproval(@Param("dto") DTO dto);
```

使用resultMapId

``` java
@Select("<script> SELECT * FROM table where id =#{id} </script> ")
@ResultMap(value = "appMap")
PO findSystemCallApprovalById(@Param("id")String id);
```

在方法 `findSystemCallApprovalById` 处报错：

> Dots are not allowed in element names，please remove it from

报错点为 org.apache.ibatis.builder.MapperBuilderAssistant 中的方法

``` java
public String applyCurrentNamespace(String base, boolean isReference) {
if (base == null) {
  return null;
}
if (isReference) {
  // is it qualified with any namespace yet?
  if (base.contains(".")) {
    return base;
  }
} else {
  // is it qualified with this namespace yet?
  if (base.startsWith(currentNamespace + ".")) {
    return base;
  }
  if (base.contains(".")) {
    throw new BuilderException("Dots are not allowed in element names, please remove it from " + base);
  }
}
return currentNamespace + "." + base;
}
```

该方法在执行时 currentNamespace=null，说明是没有找到命名空间

尝试给该Mapper.java配置Mapper.xml，显示声明命名空间，未果

尝试调整xml，interface的加载顺序，未果

继续从源码入手

使用mybatisPlus后，将Mapper.java注解解释成Mapper对象的是 

`com.baomidou.mybatisplus.MybatisMapperAnnotationBuilder`

该类是通过继承

`org.apache.ibatis.builder.annotation.MapperAnnotationBuilder`

修改其中的`parse`方法来完成的

``` java
public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
        loadXmlResource();
        configuration.addLoadedResource(resource);
        assistant.setCurrentNamespace(type.getName());
        parseCache();
        parseCacheRef();
        Method[] methods = type.getMethods();
        // TODO 注入 CURD 动态 SQL (应该在注解之前注入)
        if (BaseMapper.class.isAssignableFrom(type)) {
            GlobalConfigUtils.getSqlInjector(configuration).inspectInject(assistant, type);
        }
        for (Method method : methods) {
            try {
                // issue #237
                if (!method.isBridge()) {
                    parseStatement(method);
                }
            } catch (IncompleteElementException e) {
                configuration.addIncompleteMethod(new MethodResolver(this, method));
            }
        }
    }
    parsePendingMethods();
}
```

但是在加载方法时，并非是按照java类中的顺序加载的

> Method[] methods = type.getMethods();

这部分没有继续查下去，因为部分依赖的jar，也使用了resultMapId的方式，意味着不能从Mapper.java的方向去调整

在执行 `parseStatement(method)` 方法时，Mapper.java中的方法 `findSystemCallApprovalById` 有时会比 `findSystemCallApproval` 优先，从而导致resultMapId=appMap不识别

然后，交给

> configuration.addIncompleteMethod(new MethodResolver(this, method));

做延迟处理，但是 `MethodResolver` 类在做 `resolve()` 时，使用的方法 `parseStatement` 是有包访问权限限制的

``` java
public class MethodResolver {
  private final MapperAnnotationBuilder annotationBuilder;
  private final Method method;

  public MethodResolver(MapperAnnotationBuilder annotationBuilder, Method method) {
    this.annotationBuilder = annotationBuilder;
    this.method = method;
  }

  public void resolve() {
    annotationBuilder.parseStatement(method);
  }

}
```

MapperAnnotationBuilder中的parseStatement方法

``` java
void parseStatement(Method method) { ... }
```

MybatisMapperAnnotationBuilder虽然继承了MapperAnnotationBuilder，但是父类引用子类实例调用的是父类的受限方法`parseStatement`，子类重写此方法不算多态，导致resultMapId=appMap不识别，问题一步步跳转到MapperBuilderAssistant中去处理了

然而，受限制的方法找不到好的解决方案

最后，放弃了使用MybatisPlus的特性，老老实实写xml ...

### 新版本

在mybatisPlus-3.4.0中，` configuration.addIncompleteMethod` 方法将延迟处理的数据交给自定义的 `MybatisMethodResolver` 而不是 `MethodResolver`

没有太多的借鉴价值。

