---
title: Hexo(yilia)+Github实现相册和音频功能
comments: true
date: 2019-06-16 23:07:56
tags:
    - hexo
---

**效果图，请先点这里：https://janche.github.io/photos/**

### 1. 需要准备的资料
>**1. 本文为`hexo-theme-yilia`主题，其他hexo主题请另行百度**
>**2. GitHub上新建一个仓库存储照片（此仓库的作用除了储存还负责更新hexo博客引用的图片链接地址），为了少走弯路，请直接fork原作者的仓库(https://github.com/lawlite19/Blog-Back-Up.git)，若下载速度太慢，可选择我的备用地址(https://github.com/Janche/Blog-Photo.git)**
>**3. Python环境（安装Python3，并配置环境变量，对照片的处理是通过Python命令来处理的）**
>**4. 在你的hexo博客的`source`文件夹下(`注意不是yilia主题下的source`)，新建一个`photos`文件夹,用于存放照片相关的文件， 也可通过命令 `hexo new page photos`创建**
<!-- more -->

### 2. 更改储存照片仓库(`Blog-Photo`/`Blog-Back-Up`)中的URL
#### 2.1 仓库目录说明：
![仓库目录](https://img-blog.csdnimg.cn/20190616211830463.png)
**2.1.1 修改`ins.js`**
![修改ins.js](https://img-blog.csdnimg.cn/20190616213538210.png)
红框中的地址应改为你GitHub存储照片仓库图片的地址(`注意一定是点击download后地址栏的url`)，如图b
图a：![图1](https://img-blog.csdnimg.cn/20190616214440938.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0OTk3OTA2,size_16,color_FFFFFF,t_70)
图b:
![图2](https://img-blog.csdnimg.cn/20190616220832849.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0OTk3OTA2,size_16,color_FFFFFF,t_70)

**2.1.3 修改tool.py文件**
![修改url](https://img-blog.csdnimg.cn/20190616213043110.png)
将红框处的地址替换为hexo博客刚新建的photos地址，这样每次上传图片后将自动更新到hexo博客中。

**2.1.4 照片上传步骤**
>**1. 将`blog_photos_copy`目录下的5个文件拷贝到hexo博客刚新建的`photos`目录下，注意，只有第一次配置时需要执行这一步，后续新增照片均不需要。**
>**2. 新加的照片按照`YYYY-MM-dd_照片文字描述.jpg`格式添加到`Blog-Photo`的`photos`目录**
>**3. 运行Python命令：`python tool.py`，期间可能需要下载部分python的依赖，根据错误，百度下载就好了。** 

### 3. 配置hexo博客
#### 3.1 修改`yilia`主题的`_config.yml`
![图片3](https://img-blog.csdnimg.cn/20190616225106606.png)
#### 3.2 将empty.png图片也放于之前的`source`目录下
![empty.png](https://img-blog.csdnimg.cn/20190616231743342.png)
empty.png下载地址：https://raw.githubusercontent.com/Janche/janche.github.io/master/assets/img/empty.png
到这里，终于可以在博客上看到你自己的照片墙啦。

### 4. 为hexo博客添加音频播放
```
<!--音乐播放插件-->
<div style="margin-top:30px;">                                                       
  <iframe frameborder="no" marginwidth="0" marginheight="0" width="330" height="86" src="//music.163.com/outchain/player?type=2&id=5232465&auto=1&height=66"></iframe>
</div>
```
**你没有看错，就只有这三行代码，你想在那篇文章加音乐，直接添加这三行就行了，`src`后的链接为歌曲的外链地址，很多歌曲都因为版权而无法生成外链地址。 **

参考文章：
1. https://blog.csdn.net/u013082989/article/details/70162293
2. https://blog.csdn.net/wardseptember/article/details/82780684
