# slides库reveal.js简介

## 简介
reveal.js是一个功能强大的在线演示文稿库。尝试它的主要原因有：

* 文稿内容可以使用html标签或者markdown，可以使用任何编辑器
* 支持浏览器直接打开
* 支持使用node作为服务提供在线分享
* 支持通过chrome的打印成pdf文档

## 编辑
编写文稿比较方便，直接下载release版本的包，或者clone工程目录即可。
其中release版本可以在[https://github.com/hakimel/reveal.js/releases](https://github.com/hakimel/reveal.js/releases)
页面中选择最新版本下载即可。

基本的功能使用非常简单，可以直接编辑代码中的index.html文件，按照里面的格式编辑即可。
html文件中的```section```的标签对应slides的一页，其中按照标准的html标签编写。例如官方示例的第一页：
```html
<section>
	<h1>Reveal.js</h1>
	<h3>The HTML Presentation Framework</h3>
	<p>
		<small>Created by <a href="http://hakim.se">Hakim El Hattab</a> / <a href="http://twitter.com/hakimel">@hakimel</a></small>
	</p>
</section>
```

当然，对我来说，reveal.js最方便的地方在于可以直接在源码中使用markdown语法。markdown使用详细介绍[文档](https://github.com/hakimel/reveal.js/#markdown)，
使用时需要将markdown代码放在```script```标签中，并将type设置为```text/template```：
```html
<section data-markdown>
	<script type="text/template">
		## Markdown support

		Write content using inline or external Markdown.
		Instructions and more info available in the [readme](https://github.com/hakimel/reveal.js#markdown).
	</script>
</section>
```
这样就可以容易的将markdown文档和slides结合起来了。

## 演示和分享
slides最大的作用当然是演示了，使用reveal.js标准结构的slides演示最方便的方式，就是直接在浏览器中打开index.html。这样可以方便的在没有网络的情况下，
单机进行演示。如果需要在线分享，需要安装nodejs，然后通过node安装grunt，最后通过npm install安装reveal.js依赖。
```bash
npm install
grunt serve
```
默认启动端口为8000。

最后，一般演讲完之后，还需要分享内容。这时可以通过chrome的打印功能，直接[打印](https://github.com/hakimel/reveal.js/#pdf-export)成pdf文档。
打印时有两个地方需要特别注意：

1. 必须在访问的url上增加```print-pdf```参数。例如原url为```/index.html```，改成```/index.html?print-pdf```。
2. 打印页面中“布局”需要设置为“横向”，并且选项中勾选“背景图形”。

然后保存为pdf文件即可。
