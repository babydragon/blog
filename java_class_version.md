# 批量获取应用依赖class文件版本

## 获取class文件版本的几种方式

### javap
`javap`用于反编译java class文件。可以使用`javap -v`的方式查看class的详细信息，其中就会包含版本信息：

```
minor version: 0
major version: 52
```

### file
`file`命令通过`libmagic`来读取文件头，可以用于识别文件类型。对于java的class文件，可以使用：
```bash
$ file Application.class
Application.class: compiled Java class data, version 52.0 (Java 1.8)
```

注意，不同版本的file输出格式可能不一样。

### od
`od`命令可以用来读取二进制文件，因此理论上可以读取任何文件的任何字节。要使用`od`，首先要了解class文件头的格式：

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
从这里我们可以看见，我们关心的主版本号（major_version）在class文件开头的第六个字节开始，大小是2个字节。同时还需要注意的是，java class文件对多字节字段的存储是按照大端序（big endian）,既最高位字节存在最低的地址处。（旧版本的od不支持设置大端序和小端序，特别注意）。

了解了这些前置知识后，我们可以通过`od`命令获取class文件的版本号：

```
$ od -A n -j 6 -N 2 --endian=big -t d2 Application.class
     52
```
简单介绍下参数的含义：

* -A：设置地址基数显示方式，这里我们不需要地址基数，因此设置为n，既none，不显示地址基数。
* -j：跳过6个字节，既跳过文件头的magic和minor_version字段。
* -N：读取2个字节，既major_version字段。
* --endian：设置读取出来的字节是大端序。
* -t：设置输出数据类型为2字节的数字，这样不用再进行转换了。


## 获取工程依赖的class版本
首先最后脚本中的选型是使用`file`命令，主要原因是`javap`需要依赖jdk，`od`命令老版本不支持指定字节序，这样转换比较麻烦。从`file`输出获取版本号的脚本：

```bash
file -b ${class_file} | sed 's/.*version \(.*\)/\1/g'
```
这里使用的是服务器上老版本的`file`命令，输出没有前面提到的“(Java 1.8)”内容，因此直接抽取version后面的内容作为版本。

然后需要定位到lib目录，处理其中的jar包，首先需要解压缩jar包，然后定位到任意一个class文件，执行前面的命令。整个脚本大体为：

```bash
#!/bin/bash

APP=$1

LIB_DIR_OLD="/home/admin/$APP/target/${APP}.war/WEB-INF/lib"
LIB_DIR_NEW="/home/admin/$APP/target/${APP}/BOOT-INF/lib"

LIB_DIR=$LIB_DIR_OLD

if [[ ! -d $LIB_DIR ]]; then
    LIB_DIR=$LIB_DIR_NEW
fi

if [[ ! -d $LIB_DIR ]]; then
    echo "not find lib dir"
    exit -1
fi

cd ${LIB_DIR}
for f in *.jar
do
    tmp_dir=$(mktemp -d)
    unzip -o -q $f -d $tmp_dir
    # find first class
    class_file=$(find ${tmp_dir} -name '*.class' | head -n 1)
    if [[ -n $class_file ]]; then
        version=$(file -b ${class_file} | sed 's/.*version \(.*\)/\1/g')
        echo "$APP,$f,${version/.*}"
    fi
    rm -rf ${tmp_dir}
done
```

这里加上了解压缩到临时目录等操作。

## 批量执行
通过普通的ssh批量执行命令，我使用了`sshpass`和`parallel`两个工具。前者可以通过各种方式设置ssh密码，后者可以并行的处理。

`sshpass`可以通过环境遍历、密码文件或者命令行参数指定ssh密码，通常不建议直接在命令行里面写密码。

`parallel`命令可以解析输入文件，并行执行。我准备的数据文件是逗号分割的参数，前者为应用名，后者为ip地址。因此可以这样给`parallel`传递参数：

```bash
parallel -j 20 --colsep ',' ./get_version.sh "{1}" "{2}" < app_ip.txt
```

通过colsep参数指定输入的分隔符，这样可以将一行内容作为2个参数传递给本地需要批量执行的文件。-j参数指定并行数量，因为都是通过ssh远程执行，本地只有IO开销，这里设置的比较大。
