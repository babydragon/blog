# easyexcel导出excel

## 动态表单导出
关于easyexcel背景就不再介绍了，它对apache的POI又包装了一层，说是解决了POI的很多坑，但是文档是真的少。

首先是添加maven依赖：

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>easyexcel</artifactId>
  <version>3.1.1</version>
</dependency>
```

对于动态表单，由于无法预知结构，因此无法通过注解的方式直接定义表格结构。所以，这里需要添加表头和表格具体内容。

需要注意的是，表头的结构为`List<List<String>>`。最外面的列表表示每一个列，里面的列表是多行列头。
这种结构确保可以展示多层表头，例如有多列需要合并一个上级表头。我们的内嵌表就需要这样的结构，上层是父表格的列名，下层是内嵌表的列名。

同样，表格数据的格式也是类似的：`List<List<Object>>`，外层列表表示表格的行，内层列表表示每一列。

导出的大体代码：
```java
try (ExcelWriter writer = EasyExcel.write(exportFile)
            .registerWriteHandler(excelMergeStrategy)
            .head(headers).build()) {
  WriteSheet writeSheet = EasyExcel.writerSheet(sheetName).build();

  // prepare data
  data.forEach(d -> {
    List<List<Object>> rows;
    // expand row for embeded table
    ...
    excelMergeStrategy.reset(rows.size());

    writer.write(rows, writeSheet);
  });
}
```
注：
* exportFile: java.io.File对象，实际使用了`Files.createTempFile`方法来创建临时文件。
* excelMergeStrategy：单元格合并策略，详见下文。
* headers：`List<List<String>>`对象

由于有内嵌表的存在（既原始data数据中的一个元素，也是一个数组），因此对于原始data中的一行数据，
在处理过后，可能会变成多行（既`rows`的size是大于等于1）。因此针对每行数据，都要进行判断和展开。
同时，除了展开之外，为了最终展示表格的友好，还需要对非内嵌表的列进行合并单元格。

## 合并单元格
由于需求中的合并策略比较清晰：针对非内嵌列的数据，如果大于一行，就需要进行合并。在easyexcal中，需要自定义merge strategy：

```
public class ExcelMergeStrategy extends AbstractMergeStrategy {
    private final Set<Integer> columnNotMerge;
    private int startRowIndex;
    private int totalMergeCount;

    public ExcelMergeStrategy(Set<Integer> columnNotMerge) {
        this.startRowIndex = -1;
        this.totalMergeCount = 0;
        this.columnNotMerge = columnNotMerge;
    }

    @Override
    protected void merge(Sheet sheet, Cell cell, Head head, Integer relativeRowIndex) {
        // 如果不需要合并，或者需要合并的行小于等于1,则不需要合并
        if (columnNotMerge.isEmpty() || totalMergeCount <= 1) {
            return;
        }
        int currentColumnIndex = cell.getColumnIndex();
        if (!columnNotMerge.contains(currentColumnIndex)) {
            if (startRowIndex == -1) {
                startRowIndex = cell.getRowIndex();
            }

            int rowCount = cell.getRowIndex() - startRowIndex + 1;
            // 已经完成所有需要合并行写入，开始合并
            if (rowCount == totalMergeCount) {
                sheet.addMergedRegion(new CellRangeAddress(startRowIndex, cell.getRowIndex(), currentColumnIndex, currentColumnIndex));
            }
        }
    }

    public void reset(int totalMergeCount) {
        this.startRowIndex = -1;
        this.totalMergeCount = totalMergeCount;
    }

    public void setTotalMergeCount(int totalMergeCount) {
        this.totalMergeCount = totalMergeCount;
    }
}
```

这个合并策略比较简单，使用的时候需要注意：
1. 合并策略初始化的时候，需要根据列属性，计算出内嵌表列展开后，在excel表格中实际的列ID。（业务限制不会存在多个内嵌表）
2. 每当一个行展开流程完成后，需要调用`reset`方法，告诉这个合并策略，展开后的行数，然后再调用`write`方法写入展开后的行。

有了上述两个条件，每次执行策略的时候，就能知道当前是否已经完成所有行写入。如果已经完成，通过`reset`后第一次调用时保存的`startRowIndex`，即可计算出需要合并的单元格范围，也就是：
`CellRangeAddress(startRowIndex, cell.getRowIndex(), currentColumnIndex, currentColumnIndex)`
这个单列的范围。

## 部署
完成导出逻辑之后，本地测试没有问题，但是部署到服务器上之后，却直接在导出初始化阶段就失败了。通过catch最原始的异常，错误大致为：
```
java.lang.UnsatisfiedLinkError: /opt/java/openjdk/lib/libfontmanager.so: Shared object "libfreetype.so.6" not found
```
exec到pod内部，果然没有`libfreetype.so`这个动态链接库。

这里需要简单介绍下部署背景。我们的应用通过maven打包成jar包之后，被放入基础镜像为`eclipse-temurin:17-jre-alpine`的镜像中（为了减少部署镜像，用了基于alpine的jre），然后通过一个简单的deployment,部署在k8s集群中。

通过`ldd`命令发现，libfontmanager.so的确无法找到`libfreetype.so.6`这个动态链接库。

在alpine的[包查询页面](https://pkgs.alpinelinux.org/contents?file=libfreetype.so.6&path=&name=&branch=edge&repo=main&arch=x86_64)中，可以找到这个动态链接库是`freetype`包提供的。

尝试exec到pod中，手工通过`apk`命令安装这个包之后，导出还是失败，这次异常内容变成了：

```
Could not initialize class sun.awt.X11FontManager
```

参考[SO上的回答](https://stackoverflow.com/questions/72213414/could-not-initialize-class-sun-awt-x11fontmanager-alpine-java-17)，POI除了需要freetype之外，为了写入excel还以来字体文件。

最终需要在Dockerfile中增加相关的包安装：

```Dockerfile
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
RUN apk add --no-cache freetype fontconfig ttf-dejavu && rm -rf /var/cache/apk/*
```

注：第一行命令为了修改alpine的安装源，不然下载速度太慢了。。。

这里安装了`freetype`和`dejavu`字体，安装之后，excel就能够正常导出了。