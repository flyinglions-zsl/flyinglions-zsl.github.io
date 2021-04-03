---
title: lamda常用点记录
date: 2021-04-03 13:08:53
categories:
  - JAVA常用工具记录
tags:
  - lamda表达式

---

记录下常用的lamda语法使用

# 实体类

```java
class User{
        private Integer id;
        private String name;
        private String password;
        private String gender;
        private String phone;

        public User(){}

        public User(Integer id, String name, String password, String gender, String phone) {
            this.id = id;
            this.name = name;
            this.password = password;
            this.gender = gender;
            this.phone = phone;
        }

        public User(Integer id, String name, String phone) {
            this.id = id;
            this.name = name;
            this.phone = phone;
        }
			getXX/sexXX

        @Override
        public String toString() {
            return "User{" +
                    "id=" + id +
                    ", name='" + name + '\'' +
                    ", password='" + password + '\'' +
                    ", gender='" + gender + '\'' +
                    ", phone='" + phone + '\'' +
                    '}';
        }
    }
```

#测试数据

```java
List<User> userList = new ArrayList<>();

    @Before
    public void init(){
        User user1 = new User(1,"A","123","1","18920193281");
        User user2 = new User(2,"B","1234","1","18920194521");
        User user3 = new User(6,"C","1234","2","11920194545");
        userList.add(user1);
        userList.add(user2);
        userList.add(user3);
    }
```

# map

```java
 @Test
    public void testMap(){
        //取单个属性为list
        List<Object> nameList = userList.stream().map(User::getName).collect(Collectors.toList());
        nameList.forEach(System.out::println);
        System.out.println("-------------------");
        //取多个属性为list
        List<User> tmpList1 = userList.stream().map(user -> new User(user.getId(),user.getName(),user.getPhone())).collect(Collectors.toList());
        tmpList1.forEach(System.out::println);
        System.out.println("-------------------");
        //取两个属性转map
        Map tmpMap1 = userList.stream().collect(Collectors.toMap(User::getId,User::getName));
        tmpMap1.forEach((k,v) -> System.out.println("the key is :" + k + "; the value is :" + v));
    }
```

# foerach

```java
@Test
    public void testListForeach(){
        //循环操作数据
        userList.stream().forEach(user -> {
            if (user.getId().equals(1)){
                user.setName("重新改的");
            }
            System.out.println(user.getName());
        });
        System.out.println("-------------------");

        //循环输出
        userList.forEach(System.out::println);

        System.out.println("-------------------");
        userList.forEach(user -> System.out.println(user.getId()));
    }
```

# filter

```java
@Test
    public void testListFilter(){
        List<User> nameList = userList.stream().filter(user -> user.getId().equals(1)).collect(Collectors.toList());
        nameList.forEach(item ->{
            System.out.println(item);
        });

    }
```

# maxOrMin

```java
@Test
    public void testMaxOrMin(){
        //按某个属性取最大
        Optional<User> optionalUser1 =  userList.stream().max(Comparator.comparing(User::getId));
        if (optionalUser1.isPresent()){
            User user = optionalUser1.get();
            System.out.println(user);
        }
        //按某个属性取最小
        Optional<User> optionalUser2 =  userList.stream().min(Comparator.comparing(User::getId));
        if (optionalUser2.isPresent()){
            User user = optionalUser2.get();
            System.out.println(user);
        }
    }
```

# sorted

```java
@Test
    public void testSorted(){
        //默认升序
        List<User> sortedList1 = userList.stream().sorted(Comparator.comparing(User::getId)).collect(Collectors.toList());
        sortedList1.forEach(System.out::println);
        System.out.println("-------------------");
        //倒序
        List<User> sortedList2 = userList.stream().sorted(Comparator.comparing(User::getId).reversed()).collect(Collectors.toList());
        sortedList2.forEach(System.out::println);
        System.out.println("-------------------");
        //自定义规则sorted1.getId()-sorted2.getId()对应升序 sorted2.getId()-sorted1.getId()对应降序
        List<User> sortedList3 = userList.stream().sorted((sorted1,sorted2) -> sorted2.getId()-sorted1.getId()).collect(Collectors.toList());
        sortedList3.forEach(System.out::println);
        System.out.println("-------------------");
    }
```

# limit/reduce



```java
 @Test
    public void testLimit(){
        //从前往后取
      userList.stream().limit(2).forEach(System.out::println);
        System.out.println("-------------------");
        User user = userList.stream().limit(3).reduce(BinaryOperator.maxBy(Comparator.comparing(User::getId))).get();
        System.out.println(user);

        List<Integer> integerList = Arrays.asList(1, 2, 3, 4, 5);
        Integer sum1 = integerList.stream().reduce((sum, it) -> sum + it).get();
        System.out.println(sum1);
        //给一个初始值
        Integer sum2 = integerList.stream().reduce(1, (sum, it) -> sum + it);
        System.out.println(sum2);

        new Thread(() -> {
            System.out.println("匿名内部类");
        }).start();
    }
```

# match

```java
 @Test
    public void testMatch(){
           System.out.println(userList.stream().anyMatch(user1 -> user1.getPhone().startsWith("189")));
    }
```

