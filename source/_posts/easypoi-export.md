---
title: easypoi-export
date: 2020-07-22 14:40:33
categories:
  - tools
tags:
  - easypoi
---

用easypoi实现数据导出功能。

# 准备



基本依赖、需要配置的实体类、校验可见另一篇博文{% post_link easypoi-import easypoi-import %}

```java
@Excel(name = "客户姓名", orderNum = "1" , width = 30)
private String name; //excel: orderNum 列排序，width字段宽度
```



# 实现

```java
public class ExcelUtil {

  public static void exportExcel(List<?> list, String title, String sheetName, Class<?> pojoClass, String fileName,
                     boolean isCreateHeader, HttpServletResponse response) {
      ExportParams exportParams = new ExportParams(title, sheetName);//创建带标题和sheet名的表
      exportParams.setCreateHeadRows(isCreateHeader);
      exportParams.setStyle( ExcelExportStylerImpl.class );
      exportParams.setType( ExcelType.XSSF );
      defaultExport(list, pojoClass, fileName, response, exportParams);
    }

    private static void defaultExport(List<?> list, Class<?> pojoClass, String fileName, HttpServletResponse response,
                      ExportParams exportParams) {
      Workbook workbook = ExcelExportUtil.exportExcel(exportParams, pojoClass, list);
      if (workbook != null)
        ;
      downLoadExcel(fileName, response, workbook);
    }

    private static void downLoadExcel(String fileName, HttpServletResponse response, Workbook workbook) {
      try {
        response.setCharacterEncoding("UTF-8");
        response.setHeader("content-Type", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setHeader("Content-Disposition", "attachment;filename=" + URLEncoder.encode(fileName, "UTF-8"));
        workbook.write(response.getOutputStream());

        /*OutputStream out = response.getOutputStream();
        workbook.write( out );
        out.flush();*/
      } catch (IOException e) {
        // throw new NormalException(e.getMessage());
      }
    }

}
```



# 调用

```java
ExcelUtil.exportExcel( soptEntrustDtoList, "xxx查询明细", "xxx查询导出", XXXDto.class,format.format( new Date() ) + "xx明细.xls", true, response );

```

