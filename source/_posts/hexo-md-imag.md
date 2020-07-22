---
title: hexo中md文件图片上传
date: 2020-04-25 13:19:38
categories:
  - Hexo
tags:
  - hexo-imag
---

前言：最近用hexo和github搭建博客，老是踩坑，特别是上传图片，仅此记录下。

# 通过images文件夹

## 新建文件夹

在博客的source文件夹下新建images文件夹，放入要保存的图片

## 在md中通过github的方式引入：

```
<!-- ![任何备注](/images/xxx.jpg)(任何图片格式) -->
```

缺点就是都在一个文件夹下，不容易区分。。

# 通过插件

## 下载依赖

npm install https://github.com/CodeFalling/hexo-asset-image --save

## 修改下该插件的index.js源码

```j s
'use strict';
var cheerio = require('cheerio');

// http://stackoverflow.com/questions/14480345/how-to-get-the-nth-occurrence-in-a-string
function getPosition(str, m, i) {
  return str.split(m, i).join(m).length;
}

var version = String(hexo.version).split('.');
hexo.extend.filter.register('after_post_render', function(data){
  var config = hexo.config;
  if(config.post_asset_folder){
    	var link = data.permalink;
	if(version.length > 0 && Number(version[0]) == 3)
	   var beginPos = getPosition(link, '/', 1) + 1;
	else
	   var beginPos = getPosition(link, '/', 3) + 1;
	// In hexo 3.1.1, the permalink of "about" page is like ".../about/index.html".
	var endPos = link.lastIndexOf('/') + 1;
    link = link.substring(beginPos, endPos);

    var toprocess = ['excerpt', 'more', 'content'];
    for(var i = 0; i < toprocess.length; i++){
      var key = toprocess[i];
 
      var $ = cheerio.load(data[key], {
        ignoreWhitespace: false,
        xmlMode: false,
        lowerCaseTags: false,
        decodeEntities: false
      });

      $('img').each(function(){
		if ($(this).attr('src')){
			// For windows style path, we replace '\' to '/'.
			var src = $(this).attr('src').replace('\\', '/');
			if(!/http[s]*.*|\/\/.*/.test(src) &&
			   !/^\s*\//.test(src)) {
			  // For "about" page, the first part of "src" can't be removed.
			  // In addition, to support multi-level local directory.
			  var linkArray = link.split('/').filter(function(elem){
				return elem != '';
			  });
			  var srcArray = src.split('/').filter(function(elem){
				return elem != '' && elem != '.';
			  });
			  if(srcArray.length > 1)
				srcArray.shift();
			  src = srcArray.join('/');
			  $(this).attr('src', config.root + link + src);
			  console.info&&console.info("update link as:-->"+config.root + link + src);
			}
		}else{
			console.info&&console.info("no src attr, skipped...");
			console.info&&console.info($(this));
		}
      });
      data[key] = $.html();
    }
  }
});
```

## 站点配置文件修改

```markdown
post_asset_folder: true
```

## 确认另一个配置

```
url: https://github.com/flyinglions-zsl/flyinglions-zsl.github.io (your site)
```

## 新建md文件

```
hexo n xxxx
```

此时在_posts文件夹下生成一个xxxxmd文件及文件夹，在xxxx文件需引用的图片放入xxxx文件夹

## md中引用

```
语法：{% asset_img 图片名.格式 备注%}，这里就是url + 修改的js获取的路径 + 图片名.格式 
如：{% asset_img yuanli.jpg this %}
```

解决了在一个文件夹下的问题，且是根据年月日划分的文件夹保存的。

参考博客：

```
https://blog.csdn.net/xjm850552586/article/details/84101345
```

