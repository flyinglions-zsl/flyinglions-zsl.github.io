---
title: mybatis_study01浅识
date: 2021-01-20 19:04:33
categories:
  - mybatis
tags:
  - mybatis_study
---

# 1JDBC连接步骤回顾

```java
public static void main(String[] args) throws SQLException  {
         //1.注册数据库驱动
         DriverManager.registerDriver(new Driver());
         //2.获取数据库的连接 
         Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test?	useSSL=false", "root", "root");
         //3.创建传输器对象
         Statement  stat = conn.createStatement();
         //4.利用传输器对象传输sql语句到数据库中执行操作，将结果用结果集返回
          ResultSet rs =  stat.executeQuery("select * from book");
         //5.遍历结果集，并获取查询结果
         while(rs.next()) {
             String name = rs.getString("name");
             System.out.println(name);
         }
         //6.关闭连接（后开先关）
         rs.close(); 
         stat.close();
         conn.close();
     }    
```

