title: Java 使用 Apache POI 导出 Excel
date: '2019-04-08 14:12:50'
tags: [ApachePOI, 导出, Excel]
categories: [工作笔记]
---
## 先说点废话

&emsp;&emsp;&emsp;菜鸟做了这么久的维护工作，终于开始接触到代码编写😐，庆幸~~。“你就先做个简单的，Excel导出，没问题吧？” “额~，没，没问题”🙂。根据摸板导出Excel，那先看看摸板吧。

<!-- more -->

![](/img/20190408-1-1.png)


🤣🤣🤣怎么玩儿？百度、百度（网络上大神贴是真的强），经过一顿搜索之后，知道了Apache POI这么个东西；
&emsp;&emsp;&emsp;接触到新的东西，所以稍微整理一下，菜鸟贴，不喜勿喷；

## Apache POI 使用

&emsp;&emsp;&emsp;我的实现案例是根据自己的Excel摸板写的，不具备通用性，这边我参考了[导出任何对象集合的数据到Excel（通用版工具类）](https://layne666.site/%E5%AF%BC%E5%87%BA%E4%BB%BB%E4%BD%95%E5%AF%B9%E8%B1%A1%E9%9B%86%E5%90%88%E7%9A%84%E6%95%B0%E6%8D%AE%E5%88%B0Excel%EF%BC%88%E9%80%9A%E7%94%A8%E7%89%88%E5%B7%A5%E5%85%B7%E7%B1%BB%EF%BC%89/)，在这个的基础上进行了修改。用的是Spring boot 框架，POI版本是3.6；

### 1、 在pom.xml中导入jar包
```xml
       <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.6</version>
        </dependency>
```
### 2、实现多个sheet导出

&emsp;&emsp;&emsp;在同一个workbook下，循环遍历名称集合，通过HSSFSheet创建多个sheet; 
```java
        // 创建Excel文件
        HSSFWorkbook workbook = new HSSFWorkbook();
        // 获取名称集合
        List<TSs> ssList = exportDao.selectBhMcByAjbh(cBhAj);
        for(int i=0;i<ssList.size();i++){
        // 遍历名称
        String sheetname = ssList.get(i).getcMc();
        // 创建工作簿
        HSSFSheet sheet = workbook.createSheet(sheetname);
        ……
  
```
### 3、合并单元格

&emsp;&emsp;&emsp;首行和首列的对应的下标为0，所以在合并如：第一行、第二行，第一列、第二列单元格代码如下：
```java 
//合并单元格
CellRangeAddress callRangeAddress = new CellRangeAddress(0, 1, (short)0, (short)1);
//加载合并单元格
sheet.addMergedRegion(callRangeAddress);
```
> 合并单元格之后单元格内容会保留第一个单元格的内容；

### 4、设置行高和列宽

&emsp;&emsp;&emsp;行高和列宽的设置，有各自的计算公式（想要深入了解，度娘都有）

```java
     //设置行高为20.25
     row.setHeight((short)405);
```
> 行高计算公式：x*20 (x为需要设置的行高)

```java
     //设置列宽为37
     sheet.setColumnWidth(1,9656);//1为列号，9656为列宽
```
> 列宽计算公式：256*x+184 (x为需要设置的列宽)

### 5、自定义背景颜色
&emsp;&emsp;&emsp;我在设置背景色的时候，在度娘上找了半天，确实很多对应的标准色号。但总是感觉跟需求差很多。

```java
//自定义背景色
HSSFPalette palette = workbook.getCustomPalette();
palette.setColorAtIndex(IndexedColors.GREY_25_PERCENT.getIndex(), (byte) 214, (byte) 220, (byte) 228);

//单元格颜色
style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
style.setFillPattern(HSSFCellStyle.SOLID_FOREGROUND);
```
> IndexedColors.GREY_25_PERCENT.getIndex() 是任意找了一个色号，(byte) 214, (byte) 220, (byte) 228 根据你需要的颜色对它手动赋参数；

### 6、多行标题处理

&emsp;&emsp;&emsp;这次遇到的Excel摸板是多行标题，没想到更好的处理方法，就是将标题一下的内容存入list，用 list.size() 来确定标题一和标题二之间的间距，从而确定标题二的起始行。

### 7、输出流

&emsp;&emsp;&emsp;网上输出流的写法也各不相同，但是都大同小异。这边只不过在导出的文件名上，用了时间后缀，来防止多次导出文件名重复的问题。
```
//获取输出流
        ServletOutputStream out = null;
        try {
            out = resp.getOutputStream();
            //添加时间，防止文件名字重复
            SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdd_HHmmss");
            String timer = sdf.format(new Date(System.currentTimeMillis()));
            //设置信息头
            resp.setHeader("Content-Disposition", "attachment;filename=" + 
                           URLEncoder.encode("导出" + "(" + timer + ")", "UTF-8") + ".xls");
            resp.setHeader("Content-Type", "application/octet-stream");
            //写出文件
            workbook.write(out);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                try {
                    //10.关闭输出流
                    out.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
```

### 8、其他

&emsp;&emsp;&emsp;还有一些其他样式，使用时相对比较简单；

+ 字体位置

```
        //水平居中
        style.setAlignment(HSSFCellStyle.ALIGN_CENTER);
        //垂直居中
        style.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);
        //水平居左
        style.setAlignment(HSSFCellStyle.ALIGN_LEFT);
        ……
``` 

+ 边框线

```
        //下边框
        style.setBorderBottom(HSSFCellStyle.BORDER_THIN);
        //左边框
        style.setBorderLeft(HSSFCellStyle.BORDER_THIN);
        //上边框
        style.setBorderTop(HSSFCellStyle.BORDER_THIN);
        //右边框
        style.setBorderRight(HSSFCellStyle.BORDER_THIN);
```

+ 单元格内容自动换行

```java
        style.setWrapText(true);
```

+ 字体样式

```
        HSSFFont font = workbook.createFont();
        //设置字体大小
        font.setFontHeightInPoints(11);
        //设置字体类型
        font.setFontName("等线");
        //加载字体
        style.setFont(font);
```
##  再说点废话
&emsp;&emsp;&emsp;总结一下这次学习的感受，整个导出功能，首先是POI绘制Excel样式，这部分难点在标题二，如何准备定位到起始行，其实也是和标题一下的数据有关。所以在实现时只能先完成上半部分的导出，之后再考虑下半部分内容。再者是数据的处理，数据库查询就不提了，在插入Excel后，如何确定哪些单元格需要合并（特别是标题二下的合并）。后来也算是完成了，但是代码写的是真的烂😟。
&emsp;&emsp;&emsp;😒菜鸟的难受大神难以体会，还得继续努力、坚持、积累。💪