---
title: Hexo博客异常问题
date: 2019-08-04 21:34:20
tags: Hexo
---







本文用于记录Hexo博客搭建过程中遇到问题的解决方法，以备以后再遇到能够尽快解决；由于不是专业人员，对Hexo和前端知识也不懂，对问题解决的原理也一改不懂，见笑！！！



<!--more-->



#### 1. Hexo博客引用本地图片无法显示问题



Hexo需要将图片的路径转换到html文件中，这个操作由hexo-asset-image完成；

```
# sudo npm install https://github.com/CodeFalling/hexo-asset-image --save
```



用以下内容替换/node_modules/hexo-asset-image/index.js文件中的内容；

```
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



修改博客主目录下__config.yml文件：

```
post_asset_folder: true
```





此时，使用Hexo new "File"命令创建博客文件之后，除了在source/_post目录生成一个File.md文件之外，还会生成一个同名的File文件夹，用来保存在File.md文件中引用的本地图片；





#### 2. Hexo博客无法显示mermaid流程图问题



https://www.cnblogs.com/icoty23/p/10911231.html









