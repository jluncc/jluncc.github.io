---
title: Java 8 常用 lambda 例子总结
author: 陈加菲
date: 2020-05-18 16:02:47
tags:
 - Java
---

> Java 8 的流式计算，lambda 语法无疑是提高了对集合操作的效率，在这里我总结了一些个人常用的操作例子，并会不断更新总结。



## 集合操作

### 01 删除

#### 01 删除集合中的元素

在 Java 8 之前，删除集合（例如 List）中的某个元素，一般都是使用迭代器遍历集合中的对象，再把符合条件的元素使用 `Iterator.remove()` 删除。

```java
public static void main(String[] args) {
        User user1 = new User(5, "aa");
        User user2 = new User(2, "ba");
        User user3 = new User(3, "ac");
        User user4 = new User(1, "aad");
        User user5 = new User(4, "asd");
        // Arrays.asList()返回一个固定长度的List，不能使用流式删除集合中的元素
        // https://stackoverflow.com/questions/43020075/java-util-arrays-aslist-when-used-with-removeif-throws-unsupportedoperationexcep
        //List<User> userList = Arrays.asList(user1, user2, user3, user4, user5);
        List<User> userList = new ArrayList<>();
        userList.add(user1);userList.add(user2);userList.add(user3);userList.add(user4);userList.add(user5);

        Iterator<User> userIterator = userList.iterator();
        while (userIterator.hasNext()) {
            Integer checkId = userIterator.next().getId();
            if (checkId > 3) {
                userIterator.remove();
            }
        }

        userList.forEach(e -> System.out.println(e.getId()));
    }
```

而在 Java 8 提供的流式计算中，可以使用 `Collection.removeIf()` API 很方便的删除某个元素：

```java
public static void main(String[] args) {
        User user1 = new User(5, "aa");
        User user2 = new User(2, "ba");
        User user3 = new User(3, "ac");
        User user4 = new User(1, "aad");
        User user5 = new User(4, "asd");
        // Arrays.asList()返回一个固定长度的List，不能使用流式删除集合中的元素
        // https://stackoverflow.com/questions/43020075/java-util-arrays-aslist-when-used-with-removeif-throws-unsupportedoperationexcep
        //List<User> userList = Arrays.asList(user1, user2, user3, user4, user5);
        List<User> userList = new ArrayList<>();
        userList.add(user1);userList.add(user2);userList.add(user3);userList.add(user4);userList.add(user5);

        userList.removeIf((User u) -> u.getId() > 3);
        userList.forEach(e -> System.out.println(e.getId()));
    }
```



### 02 遍历

#### 01 按集合中某个元素分组

例如，我们有一个集合，里面包括学生报课的信息，我们需要将这个集合按照学生 ID 进行分组转换成 Map。在 Java 8 之前，我们需要遍历一次集合，判断元素是属于哪个学生 ID：

```java
public static void main(String[] args) {
        // User{id, name, clazzId}
        User user1 = new User(1, "aa", 100);
        User user2 = new User(1, "aa", 200);
        User user3 = new User(2, "bb", 100);
        List<User> userList = new ArrayList<>();
        userList.add(user1);userList.add(user2);userList.add(user3);

        Map<Integer, List<User>> userId2UserClazzsMap = new HashMap<>();
        for (User user : userList) {
            if (userId2UserClazzsMap.get(user.getId()) == null) {
                List<User> userSet = new ArrayList<>();
                userSet.add(user);
                userId2UserClazzsMap.put(user.getId(), userSet);
            } else {
                List<User> userSet = userId2UserClazzsMap.get(user.getId());
                userSet.add(user);
                userId2UserClazzsMap.put(user.getId(), userSet);
            }
        }
        userId2UserClazzsMap.forEach((key, value) -> System.out.println(key + "---" + value.size()));
    }
```

而在 Java 8 中，我们可以很方便的使用 `Collectors.groupingBy()` API 来实现相同功能：

```java
public static void main(String[] args) {
        // User{id, name, clazzId}
        User user1 = new User(1, "aa", 100);
        User user2 = new User(1, "aa", 200);
        User user3 = new User(2, "bb", 100);
        List<User> userList = new ArrayList<>();
        userList.add(user1);userList.add(user2);userList.add(user3);

        Map<Integer, List<User>> userId2UserClazzsMap = userList.stream().collect(Collectors.groupingBy(User::getId));
        userId2UserClazzsMap.forEach((key, value) -> System.out.println(key + "---" + value.size()));
    }
```

此外需要注意的一点是，groupingBy() 方法返回的是一个 List，如果需要去重元素，存储进 Set，还需要额外的操作。

#### 02 List 与 Set 的互换

**List to Set**

使用上一个例子，如果要将 userList 转换成 Set，在 Java 8 之前，可以使用 HashSet 构造函数：

```java
Set<User> userSet = new HashSet<>(userList);
```

或者笨方法，遍历 userList，逐一添加进 Set 集合中。

而在 Java 8 的流式计算中，也提供了 List 转 Set 的快捷方法：

```java
Set<User> userSet = userList.stream().collect(Collectors.toSet());
```

**Set to List**

类比 List to Set，反过来，将 Set 转换成 List，在 Java 8 之前，同样有 ArrayList 构造函数，或者遍历添加进集合：

```java
List<User> users = new ArrayList<>(userSet);
```

Java 8 提供的流式计算，同样提供了类似的转化 API：

```
List<User> users = userSet.stream().collect(Collectors.toList());
```



### 参考文章

[Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/)



**TO BE CONTINUE**