﻿# 使用Flying Saucer生成pdf

> 实现思路：
> 利用FreeMarker编写HTML模板，通过Flying Saucer生成pdf。

## 一、maven添加依赖 ##
#### 本项目使用maven构建，基于spring架构

在pom.xml中添加

	<dependency>
			<groupId>org.xhtmlrenderer</groupId>
			<artifactId>flying-saucer-pdf</artifactId>
			<version>9.0.9</version>
	</dependency>

它的依赖关系图为

![依赖关系图](http://highness.qiniudn.com/pom%E5%85%B3%E7%B3%BB%E4%BE%9D%E8%B5%96%E5%9B%BE.png)

-------------------------------------------------
## 二、用FreeMarker编写模板 ##
1. 在webroot目录下新建一个pdfTemplate的文件夹

在里面新建一个freemarker的模板文件，命名为： template.ftl
其中头部代码要使用标准的html文档声明，否则iText会解析失败。

    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">  
	<html xmlns="http://www.w3.org/1999/xhtml">
	<head>
	<title>${title}</title>
	<meta http-equiv="content-type" content="text/html;charset=utf-8"></meta>
	<style type="text/css">	
		body {
			font-family: SimSun;
		}
	</style>
	</head>
	<body>
	
	</body>
	</html>

#### 需要注意的几点：
1. HTML页面必须是标准的XHTML页面
2. 所有的html标签必须使用小写字母，同时也要使用闭合写法，因为Flying Saucer按xml语法解析
3. 需统一文件编码，避免乱码的出现
4. 中文乱码的问题和css样式文件引入还有图片资源引入的问题，这个下面单独讲
5. 页面里的js代码都将不起作用
6. 表格不支持thead标签



--------------------------------   
## 三、web.xml中的配置 ##
在web.xml中添加如下映射

    <servlet>  
    <servlet-name>exportPDF</servlet-name>  
    <servlet-class>com.tg.servlet.ExportPDFServlet</servlet-class>  
	</servlet>  
	<servlet-mapping>  
    <servlet-name>exportPDF</servlet-name> 
    <url-pattern>/exportPDF/*</url-pattern> 
	</servlet-mapping>

浏览器下载地址为： http://localhost:8080/exportPDF/downloadpdf
其中，downloadpdf可以自定义映射，也可以跟模板名称相关联，这样可以实现下载指定模板的需求。

-------------------------------
## 四、后台Servlet代码 ##
后台ExportPDFServlet.java的代码如下：

    package com.tg.servlet;

	import java.io.IOException;
	import java.io.OutputStream;
	import java.io.StringWriter;
	import java.util.HashMap;
	import java.util.Locale;
	import java.util.Map;
	
	import javax.servlet.ServletException;
	import javax.servlet.http.HttpServlet;
	import javax.servlet.http.HttpServletRequest;
	import javax.servlet.http.HttpServletResponse;
	import javax.xml.parsers.FactoryConfigurationError;
	
	import org.xhtmlrenderer.pdf.ITextRenderer;
	
	import com.lowagie.text.DocumentException;
	import com.lowagie.text.pdf.BaseFont;
	
	import freemarker.template.Configuration;
	import freemarker.template.Template;
	import freemarker.template.TemplateException;
	
	public class ExportPDFServlet extends HttpServlet {
		/**
		 * 
		 */
		private static final long serialVersionUID = 1L;
		// 负责管理FreeMarker模板的Configuration实例
		private Configuration cfg = null;
	
		public void init() throws ServletException {
			// 创建一个FreeMarker实例
			cfg = new Configuration();
			// 指定FreeMarker模板文件的位置
			cfg.setServletContextForTemplateLoading(getServletContext(), "/pdfTemplate");
		}

		public void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
			
			ITextRenderer renderer = new ITextRenderer();
			try {
				String realPath = getServletContext().getRealPath("/");
				// 对应css中字体样式 ： font-family:SimSun
				renderer.getFontResolver().addFont(realPath + "/pdfTemplate/simsun.ttc", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);
				// 对应css中字体样式 ： font-family:SimHei
				renderer.getFontResolver().addFont(realPath + "/pdfTemplate/SIMHEI.TTF", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);
				
				// 建立数据模型
				Map<String, Object> data = new HashMap<String, Object>();
				data.put("title", "导出pdf模板");
				data.put("name", "数据");
				
				String html = null;
				cfg.setEncoding(Locale.CHINA, "UTF-8");
				// 获取模板文件 template.ftl
				Template template = cfg.getTemplate("template.ftl","UTF-8");
				template.setEncoding("UTF-8");
				StringWriter writer = new StringWriter();
				// 将数据输出到html中
				template.process(data, writer);
				writer.flush();
				html = writer.toString();
				// 
				renderer.setDocumentFromString(html);
				
				// 解决图片的相对路径问题  ##必须在设置document后再设置图片路径，不然不起作用
		        renderer.getSharedContext().setBaseURL(realPath + "/images/");
				renderer.layout();
				
				/*
				 * 设置浏览器下载响应
				 */
				response.reset();  
		        response.setContentType("application/pdf");  
		        response.setContentType("application/force-download");
		        String fileName = "导出测试.pdf";
				fileName = new String(fileName.getBytes("gb2312"), "iso8859-1");
				response.addHeader( "Content-Disposition","attachment; filename=" +fileName) ;
		        OutputStream out = response.getOutputStream();  
				renderer.createPDF(out, false);
				renderer.finishPDF();
		        out.flush();
				out.close();
				
			} catch (DocumentException | FactoryConfigurationError e1) {
				// TODO Auto-generated catch block
				e1.printStackTrace();
			} catch (TemplateException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}

		public void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
			doPost(request, response);
		}
	
		public void destroy() {
			super.destroy();
		}

	}

#### 代码说明及注意事项：
+  `// 指定FreeMarker模板文件的位置 `
    `cfg.setServletContextForTemplateLoading(getServletContext(), "/pdfTemplate");`
   这里只需指定模板存放的目录即可，不需要具体到某一个模板文件

+ **中文支持**  这个是非常重要的，需要去 `C:/Windows/Fonts/` 目录下找字体文件，如果是window系统可以这样直接指定字体文件地址。
	`renderer.getFontResolver().addFont("C:/Windows/Fonts/simsun.ttc", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);`
这里为了方便，我把`C:/Windows/Fonts/` 目录下的字体文件拷贝到了项目pdfTemplate目录下，共有两个，一个是`宋体`（simsun.ttc），一个是`黑体`（SIMHEI.TTF）。
**注意** 在模板里也要同样指定中文字体，样式代码如下：
```
	body {
		font-family: SimSun;
	}
```
建议在body上设置字体样式，这样可以避免遗漏。

+ `// 获取模板文件 template.ftl`
  `Template template = cfg.getTemplate("template.ftl","UTF-8");`
在这里指定了模板文件，可以把模板文件名称和下载请求路径关联起来，实现动态获取某一个模板

+ 图片显示的问题，其实就是图片路径的问题，Flying Saucer不支持相对路径，所以这里使用了`renderer.getSharedContext().setBaseURL(realPath + "/images/");`的绝对路径； 经测试，其实使用网络路径也是可以的，例如：`<img src="${basePath}/images/head.png" />`

+ freeMarker接收数据 `template.process(data, writer);` 其中第一个参数为Object类型，所以建议传递一个Map对象，在里面用key-value 的形式传输各种类型的数据


    

-----------------------
## 五、编写模板注意事项 ##
- **引入外部的css文件**，注意`link`标签要闭合写法
`<link href="${basePath}/style.css" rel="stylesheet" type="text/css"></link>`
或者使用绝对物理路径
`<link href="file:///C:/PDF/style.css" rel="stylesheet" type="text/css"></link>`

- **添加水印**，实现方式非常简单，首先准备一张水印图片，然后在样式中设置`body`背景即可，例如：
	
```
    body {
		font-family: SimSun;
		background:url(${basePath}/水印图片.png) no-repeat fixed center;
	}
```


- **页眉 页脚** 在`body`中任意位置添加如下代码：
```
	   <div id="header">
	    <div id="header-left" class="header-left" align="left">页眉左</div>
	    <div id="header-center" class="header-center" align="center">页眉中间</div>
	    <div id="header-right" class="header-right" align="right">页眉右</div>
	</div>
	<div id="footer">
	    <div id="footer-left" class="footer-left" align="left">页脚左</div>
	    <div id="footer-center" class="footer-center" align="center">页脚中间</div>
	    <div id="footer-right" class="footer-right" align="right">
	    	第 <span id="pagenumber"/> 页，共 <span id="pagecount"/> 页
	    </div>
	</div>
```
然后在`<style>`样式中添加如下代码：
```
	div.header-left {display: none}
	div.header-center {display: none}
	div.header-right {display: none}
	div.footer-left {display: none}
	div.footer-center {display: none}		     
	div.footer-right {display: none}		     

	@media print {
		div.header-left {
	        display: block;
	        position: running(header-left);
	    }
	    div.header-center {
	        display: block;
	        position: running(header-center);
	    }
	    div.header-right {
	        display: block;
	        position: running(header-right);
	    }
	    div.footer-left {
	        display: block;
	        position: running(footer-left);
	    }
	    div.footer-center {
	        display: block;
	        position: running(footer-center);
	    }
	    div.footer-right {
	        display: block;
	        position: running(footer-right);
	    }
	}	

	@page {       
	      size: 8.5in 11in; 
	      margin: 0.65in;
	      padding: 1em; 
	      @top-left{
	        content:element(header-left);
	      };
	      @top-center {     
		    content: element(header-center);     
	      };
	      @top-right {     
		    content: element(header-right);     
	      };
	      @bottom-left { 
	      	content: element(footer-left) 
	      };  
	      @bottom-center {     
		    content: "page " counter(page) " of  " counter(pages) ;     
	      };     
	      @bottom-right { 
	      	content: element(footer-right) 
	      }; 
	} 

	#pagenumber:before {  
	    content: counter(page);  
	}  
	  
	#pagecount:before {  
	    content: counter(pages);  
	}
```
至此，页眉页脚的代码便全部写完了，可以根据实际需要去掉不需要的页眉页脚。

- **强制分页** 假如需要在某一个地方分页，可以用下面的方法实现，首先在`style`样式定义中添加一行代码  `.pageNext{page-break-after: always;} `
然后在页面需要分页的位置加上如下代码即可。
`<div class='pageNext'></div>` 

- **目录书签** pdf的书签实现其实是用了html的锚点原理，即html中添加锚点代码：
	```
    <a name="aaa">锚点一</a>
	... ... 
	<a name="bbb">锚点二</a>
	```
然后在`</head>`标签里添加如下代码：

```
<bookmarks>  
  	<bookmark name="书签一" href="#aaa" /> 
  	<bookmark name="书签二" href="#bbb" /> 
</bookmarks>
```



## **注** ##
1. FreeMarker相关语法文档

- [Apache FreeMarker Manual](http://freemarker.org/docs/)

- [FreeMarker基本语法知识 - 技术年代 - 博客频道 - CSDN.NET](http://blog.csdn.net/wu209000/article/details/4222129)

2. 其它






















































































