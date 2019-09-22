---
title: Spring XML 配置文件中命名空间的理解
author: 陈加菲
date: 2019-02-17 19:28:47
tags:
 - 其他
---

### 命名空间的理解

首先我们需要对「命名空间」有基本的理解。

#### 例子

命名空间一般放置在 XML 文件的开始标签中，如下最基本的 Spring XML 配置文件中，xmlns / xmlns:xsi / xsi:schemaLocation 三项便是命名空间。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">

</beans>
```

#### 作用

简单来说，命名空间的作用是**自动为 XML 标签补全前缀，避免标签命名冲突** （因为开发者可自定义的标签名，存在冲突可能）。

譬如，Spring 的 XML 配置文件，配置项都在 <beans\> 标签中，但这时开发者如果自定义一个自己的标签，同样命名为 <beans\>，便造成了冲突，XML 解析器无法分辨这些冲突的命名。

此时命名空间便派上用场了。在 <beans\> 开始标签中添加命名空间，实际上 XML 解析器会**自动为该标签名补全前缀**，以区别其他的标签，避免了命名的冲突。


### Spring XML 配置文件中的命名空间

有了上面的背景知识，我们来理解 Spring 的 XML 配置文件中命名空间的作用，也是大致相同：避免标签命名冲突。

除此之外，我们再来看看 Spring 中的命名空间，见上面例子。
可以发现在最基本的配置中，含有 xmlns, xmlns:xsi, xsi:schemaLocation 三项，这三项是 Spring 最基本的命名空间，其含义如下：

1. xmlns 表示是该 XML 文件的默认命名空间；
2. xmlns:xsi 表示该 XML 文件遵守 xml 规范；
3. xsi:schemaLocation 表示具体用到的 schema 资源。

因此在我们需要引入新的 Spring 配置时，添加一个新的命名空间，以及其 schema 资源便可，如下引入 Spring 上下文配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.0.xsd">

    <!-- 扫描并自动装配bean -->
    <context:component-scan base-package="com.demo" />

</beans>
```

### 参考文章

[Spring 配置文件之引入命名空间](https://blog.csdn.net/hhx_echo/article/details/76095840)
[XML 命名空间](http://www.w3school.com.cn/xml/xml_namespaces.asp)

**END**

