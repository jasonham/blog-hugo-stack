---
title: 怎样给hugo加上用户统计功能
description: 
date: 2023-05-28T09:31:18+08:00
lastmod: 2023-05-28T09:31:18+08:00
slug: how-to-use-baidu-analytics-on-hugo

tags:
 - Hugo
 - Stack
categories:
 - Web
---
百度统计是一款功能强大的数据分析工具，可以帮助您更好地了解您的网站或应用程序的用户行为。它提供了丰富的数据分析工具和报告，包括网站流量、用户行为、Conversions等。

要使用百度统计需要先到[tongji.baidu.com](https://tongji.baidu.com/) 进行注册，并且添加站点，然后获取一个script的脚本，并将这个脚本放置到网站的HTML的HEAD部分，在hugo中提供了layouts目录用于替换模板的默认页面，我们需要按Stack模板主题的结构新建一个head文件夹，并在下面添加一个custom.html的文件。  
目录结构如下：  
```mermaid
/root
 └─layouts
    └─head
       └─custom.html

```
在custom.html文件中粘帖上script脚本即可。  
等待十多分钟脚本生效后，就可以在[百度统计](https://tongji.baidu.com/)看到分析效果。

