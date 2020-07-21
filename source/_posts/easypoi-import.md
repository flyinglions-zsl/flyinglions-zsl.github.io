---
title: easypoi-import
date: 2020-07-21 10:31:27
categories:
  - tools
tags:
  - easypoi
---

前言：最近需要做一个文件导入的功能，无需太复杂，于是想起了easypoi，仅此记录下实现过程。(基于springboot)

# 依赖

```java
			  <dependency>
            <groupId>cn.afterturn</groupId>
            <artifactId>easypoi-base</artifactId>
            <version>3.0.3</version>
        </dependency>
        <dependency>
            <groupId>cn.afterturn</groupId>
            <artifactId>easypoi-web</artifactId>
            <version>3.0.3</version>
        </dependency>
        <dependency>
            <groupId>cn.afterturn</groupId>
            <artifactId>easypoi-annotation</artifactId>
            <version>3.0.3</version>
        </dependency>
```



# 创建实体类

用过esaypoi的都清楚，必须有个实体类来接收相关字段数据。

对照导入excel的模版，建立导入实体类(需实现IExcelModel接口)：

{% asset_img excel模版.png %}

```java
--importEntity
@Excel(name = "客户姓名", orderNum = "1")
private String name;
@Excel(name = "客户编号", orderNum = "0")
private String clientId;
@Excel(name = "营业部", orderNum = "2")
private String branchName;
@Excel(name = "激励计划", orderNum = "3")
private String jljhName;
```



对于特殊的可以确定类型的字段：

```java
@Excel(name = "解锁期", replace = {"一期_1", "二期_2","三期_3"}
private String jsq;
       
@ExcelCollection(name = "期权信息")
private List<QqEntity> qqList;
--此处QqEntity同上面
       
@Excel(name = "操作日期", exportFormat = "yyyy-MM-dd")//导入
private Date operateDate;
```



# 编写工具类

```java
public static <T> ExcelImportResult<T> importExcel(MultipartFile file, Integer titleRows, Integer headerRows,
			Class<?> pojoClass,IExcelVerifyHandler<?> excelVerifyHandler) throws IOException, Exception {  //IExcelVerifyHandler<?> excelVerifyHandler 与校验有关，看个人，后面讲解
		if (file == null) {
			return null;
		}
		ImportParams params = new ImportParams();
	  //params.setTitleRows(titleRows); 表标题，没有默认0
		//params.setHeadRows(headerRows); 涉及表头文字占用几行
		params.setNeedVerfiy(true); //导入数据的校验
		params.setVerifyHanlder(excelVerifyHandler); //设置自定义handler进行校验
		ExcelImportResult<T> list = null;
		try {
			list = ExcelImportUtil.importExcelMore(file.getInputStream(), pojoClass, params);
		} catch (NoSuchElementException e) {
			throw new NormalException("excel文件不能为空");
		} 
		return list;
	}
```



# 数据校验

数据的基本校验可以由两种方式实现：

## 注解

EasyPOI 支持 Hibernate Validator ,于是可以采用注解来实现;不过基本是写比较简单的判断。

```java
@Excel(name = "客户编号", orderNum = "0")
@NotBlank(message = "[客户编号]不能为空")
private String clientId;

@Excel(name = "解锁期", replace = {"一期_1", "二期_2","三期_3"}
@Pattern(regexp = "[123]", message = "期数错误")
private String jsq;
       
private static final String date_pattern = "yyyy-MM-dd";
@Excel(name = "操作日期")
@Pattern(regexp = date_pattern, message = "日期格式错误")
private Date operateDate;
```



## 自定义handler类

如需实现复杂的判断逻辑，可以通过实现IExcelVerifyHandler 接口，重写verifyHandler方法来计算、判断。

```java
public class QqIExcelVerifyHandler implements IExcelVerifyHandler {
	private QqDao dao;
		
	@Override
	public ExcelVerifyHanlderResult verifyHandler(Object obj) {
		......
	}
}
```

此处即可说明为何编写工具类多传一个**IExcelVerifyHandler** 参数。 且返回类型是ExcelVerifyHanlderResult



## 源码跟进看下基本思路

```java
1.importExcelMore 进去
2.importExcelByIs 子方法进入
 此时方法有两个list： 
   private List<T> list; //用来存储无误的数据
   private List<T> failList; //用来存储出错的数据
3.result.addAll(this.importExcel(result, book.getSheetAt(i), pojoClass, params, pictures));
4.importExcel 进去
  4.1获取所有带@Excel注解的feilds，也就是导入excel的表头对应字段
  4.2做一些表字段顺序、是否缺失的校验
  4.3其实首先会判断ExcelCollectionParams，猜测是@ExcelCollection注解相关
  4.4	if (this.verifyingDataValidity(object, row, params, pojoClass)) {
    collection.add(object);
  } else {
    this.failCollection.add(object);
  }
  此处调用自定义handler的 verifyHandler方法进行校验 (此时传的参数就是 自定义的导入entity对象)
   4.5在verifyHandler方法进行校验方法中，根据业务条件判断，最终可以遍历拼接错误msg
   4.6此处进行无误和出错数据List的装载
  
```

