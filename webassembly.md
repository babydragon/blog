# 初识webassembly

Chrome 57正式发布，带来了对webassembly的正式支持，因此决定尝试下webassembly。从webassembly标准来看，它主要是将web上的语言，从解释型的javascript，变成了已经编译从AST（抽象语法树），以减少js引擎的解析编译时间。

## 安装Emscripten
按照webassembly[网站介绍](http://webassembly.org/getting-started/developers-guide/)，webassembly需要依赖Emscripten工具来将源码（本文使用简单的C代码）编译成wasm文件。[Emscripten](https://github.com/kripken/emscripten)在最近使用的一个[viz.js](https://github.com/mdaines/viz.js/)工程中看见过。它的主要作用是通过llvm前端将源码编译成AST之后，再转换成js文件。viz.js工程将整个[graphviz](http://www.graphviz.org/)编译成了js，我们用它来直接在浏览器中解释dot文件，并展示成svg图片。

Emscripten安装最初参照了http://webassembly.org/getting-started/developers-guide/ 中描述的方式，但是发现其安装包基本放在S3上，网络不畅，另外下载的node是4.x版本，最终使用时还直接报错退出，最终还是直接找了一个overlay安装。

以下仅针对gentoo用户：

```bash
layman -a science
emerge dev-util/emscripten
```

安装之后，主要使用其提供的emcc，em++等命令，用来替换gcc、g++。

## 简单的例子
安装完Emscripten之后，就可以开始尝试第一个webassembly程序。首先先用最简单的C来实现一个整数加法：

```c
#include <emscripten/emscripten.h>

#ifdef __cplusplus
extern "C" {
#endif

int EMSCRIPTEN_KEEPALIVE add(int a, int b) {
	return a + b;
}

#ifdef __cplusplus
}
#endif
```

代码非常简单，这里唯一需要注意的是```EMSCRIPTEN_KEEPALIVE```宏，该宏的主要作用是防止该函数被编译器优化而无法导出，具体请参见[Emscripten FAQ](https://kripken.github.io/emscripten-site/docs/getting_started/FAQ.html#why-do-functions-in-my-c-c-source-code-vanish-when-i-compile-to-javascript-and-or-i-get-no-functions-to-process)。当然，随着```EMSCRIPTEN_KEEPALIVE```宏引入，还需要引入```emscripten/emscripten.h```头文件。

然后我们在bash中执行：
```bash
emcc -s WASM=1 -O3 -o add.js add.c
```
此时，emcc会生成```add.asm.js add.js  add.wasm```三个文件，其中add.asm.js文件是asm的fallback，当浏览器不支持webassembly的时候可以用此方式执行，这里我们不去使用。add.wasm是代码生成的二进制文件，add.js是Emscripten提供的脚手架js，用于读取和解析对应的wasm文件。

ps：如果输出文件改成add.html，Emscripten会生成可以直接使用的html文件，但是这个示例没有main函数，因此也无法直接看见效果。

我们没有使用Emscripten生成的html，因此需要手工写一个。

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>test wasm</title>
    </head>
    <body>
        <input type="number" id="left" value="0">
        <input type="number" id="right" value="0">
        <br/>
        <button id="add" name="button">calc</button>

        <script type="text/javascript">
            var Module = {};
            fetch('add.wasm').then(response => response.arrayBuffer())
            .then((bytes) => {
                Module.wasmBinary = bytes;
                var script = document.createElement('script');
                script.src = "add.js";
                document.body.appendChild(script);
            });

            document.getElementById('add').addEventListener('click', () => {
                var leftValue = document.getElementById('left').value;
                var rightValue = document.getElementById('right').value;
                var result = Module.ccall('add', 'number', ['number', 'number'], [leftValue, rightValue]);
                alert(result);
            })
        </script>
    </body>
</html>
```

下面简单说明下，HTML本身很简单，用了两个input框，和一个按钮。按钮逻辑很简单，获取两个input框中的数字，然后调用wasm提供的add函数，并将结果通过alert框弹出。

然后是js代码：
```javascript
var Module = {};
fetch('add.wasm').then(response => response.arrayBuffer())
.then((bytes) => {
    Module.wasmBinary = bytes;
    var script = document.createElement('script');
    script.src = "add.js";
    document.body.appendChild(script);
});
```
这里主要参照Emscripten生成的html中加载wasm的方式，首先通过fetch方式将wasm文件加载到浏览器，加载完成后，将其中的二进制字节数组赋值给Module.wasmBinary对象。最后通过创建一个script标签来加载脚手架js：add.js。

这里需要特别注意，add.js加载的时候就会去解析Module.wasmBinary中的数据，因此必须在js加载前，完成前面两步，既加载wasm文件和初始化Module.wasmBinary对象。

调用方式非常简单，add.js对外提供了```ccall```和```cwrap```两个函数，第一个用于直接调用c函数，第二个用于将c函数包装成标准js函数，以便后续调用。这里直接在按钮的click事件中，通过```ccall```函数来调用c代码中的```add```函数，实现两个整数相加：
```javascript
var result = Module.ccall('add', 'number', ['number', 'number'], [leftValue, rightValue]);
```

```ccall``` 函数有4个参数，第一个参数是函数名称，这里和C代码中的相同，**注意：**如果是C++代码，建议加上```extern "C"```块，避免因为函数名称混淆导致无法获取。第二个参数是返回值类型；第三个参数是参数类型，以数组方式，按照参数顺序添加；最后一个参数是函数实际的参数。

## 坑
按照惯例，第一次尝试不会顺风顺水，以下是遇到的一系列问题：

### 安装问题
Emscripten安装遇到的问题前面已经提到了，一个是亚马逊aws s3服务不稳定，导致包下载了半天。还有就是安装之后，默认下载的node有问题，导致无法使用。最后还是通过overlay中的ebuild文件安装。

### wasm加载问题
因为Chrome 57已经原生支持wasm文件，最初加载的时候，尝试抛弃Emscripten提供的add.js，直接通过webassembly[原生接口](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiate)进行加载。结果发现```WebAssembly.instantiate```函数第二个参数importObject总是设置不好，导致无法正常导入。查了半天文档，最后发现Emscripten目前版本，还不支持生成独立的wasm文件，必须搭配其生成的脚手架js一起使用（或者模仿其加载方式）。[独立模式](https://github.com/kripken/emscripten/wiki/WebAssembly-Standalone)还在开发中，因此没有尝试。

加载wasm遇到的另外一个问题，是v8引擎加载wasm文件时内存溢出。刚开始使用```emcc```命令编译C文件时，没有增加-O参数（其作用和gcc的-O参数相同，设置编译器优化级别），结果Chrome在加载时，会提示：

> Wasm compilation exceeds internal limits in this context for the provided arguments

导致C函数无法导入。这个问题网上搜索了半天也没找到解决方案，搜这个文案的结果，基本都是V8源码。。。当时都已经快放弃Chrome，打算安装一个Firefox试试了。最终尝试了增加```-O3```修改了优化级别，Chrome可以正常加载wasm文件了。具体原因未明。
