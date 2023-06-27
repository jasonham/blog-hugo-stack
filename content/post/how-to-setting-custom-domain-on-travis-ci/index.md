---
title: 如何在travis ci自动部署时设置自定义域名

date: 2022-06-27T19:23:40+08:00
updated: 2023-06-27T19:23:40+08:00
slug: how-to-setting-custom-domain-on-travis-ci

tags:
 - Travis
 - Github
categories:
 - CI
---
在Travis CI部署Github pages时，默认是不会生成自定义域名的，每次都需要去Github的setting里重新设置一下。
如果需要在部署时自动配置好自定义域名，可以在 .travis.yml的deploy内添加

``` yml .travis.yml
...
deploy:
  fqdn: [你的域名]
...
```