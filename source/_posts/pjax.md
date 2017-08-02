title: pjax简要分析
author: libaogang
tags:
  - pjax
  - ajax
categories:
  - 技术
thumbnail: /img/random/material-8.png
date: 2017-07-30 13:12:00
---
>在寻找博客程序的过程中了解到了pjax技术，本文简要介绍了pjax技术。遗憾的是，由于本博客架构设计，不能以很小的代价切换到pjax技术。可以移步该[博客](https://tale.biezhi.me)体验下丝滑般的站内页面跳转。

可以观察到github，facebook都支持这样的一种浏览方式， 当你点击一个站内的链接的时候， 不是做页面跳转， 而是只是站内页面刷新。 浏览器地址栏位上面的地址也会更改， 用浏览器的回退功能也能够回退到上一个页面。这样的用户体验， 比起整个页面都闪一下来说好很多。通过观察请求可以发现，他们都用到了pjax技术。



## history新接口
 pjax技术使用到了html5中history对象的新方法，简要介绍如下：
- pushState(state,title,url)方法

向history对象中添加URL。pushState()有三个参数:state对象是一个JSON对象（对应URL，唯一性),它关系到由pushState()方法创建出来的新的history实体；title(现在是被忽略，未作处理)；URL(请求的链接)。利用该方法改变url而不会触发页面的刷新。

- replaceState(state,title,url)方法

-history.replaceState() 替换当前的URL而不产生历史记录，参数和pushState一样。

- popstate事件

当history实体被改变时，popstate事件将会发生。如果history实体是有pushState和replaceState方法产生的popstate事件的state属性会包含一份来自history实体的state对象的拷贝。

## 传统ajax
虽然传统ajax可以异步获取服务端数据并无刷新改变页面内容，但是无法改变浏览器url。因此有种方案是在内容发生改变后通过改变URL的hash的方式获得更好的可访问性（因为只有改变url的hash才不会触发页面刷新），但是这种方案有时候不能很好的处理浏览器的前进和后退。pjax的出现就是为了解决这些问题。
## pjax
pjax就是pushState和ajax的结合，不需要重新加载整个页面就能从服务器加载Html到你当前页面，这个ajax请求会有永久链接、title并支持浏览器的回退/前进按钮。

### 优点
- 减轻服务端压力
按需请求，每次只需加载页面的部分内容，而不用重复加载一些公共的资源文件和不变的页面结构，大大减小了数据请求量，以减轻对服务器的带宽和性能压力，还大大提升了页面的加载速度。
- 优化页面跳转体验
常规页面跳转需要重新加载画面上的内容，会有明显的闪烁，而且往往和跳转前的页面没有连贯性，用户体验不是很好。如果再遇上页面比较庞大、网速又不是很好的情况，用户体验就更加雪上加霜了。使用pjax后，由于只刷新部分页面，切换效果更加流畅，而且可以定制过度动画，在等待页面加载的时候体验就比较舒服了。

### 缺点
- 使服务端处理变得复杂
要做到普通请求返回完整页面，而pjax请求只返回部分页面，服务端就需要做一些特殊处理，当然这对于设计良好的后端框架来说，添加一些统一处理还是比较容易的，自然也没太大问题。另外，即使后台不做处理，设置pjax的fragment参数来达到同样的效果。

### 简易原理

观察github的请求,当点击Issues标签时，浏览器发送了一个ajax请求，拉取了部分html片段，同时浏览器url及标题也发生变化。
![](/image/pjax.jpg)

以下代码实现了一个简易pjax以帮助理解pjax原理
```javascript
 $(document).ready(function() {
        $('#main').on('click','a',function(e) {
            if(window.history.pushState) {
                e.preventDefault(); //拦截链接跳转
                url = $(this).attr('href');
                $.ajax({
                    async: true,
                    type: 'GET',
                    url: 'data.php',
                    data: 'pjax=1',
                    success: function(data) {
                        window.history.pushState(null, null, url); //改变URL和添加返回历史
                        document.title = data.title; //设置标题
                        $('#main').html(data.main); //更换页面main部分内容
                    }
                });
            } else {
                return; //低版本IE8等不支持HTML5 pushState,直接返回进行链接跳转
            }
        });
    });
```


### 项目地址
pjax源码及具体的插件使用方式见[jquery-pjax](https://github.com/defunkt/jquery-pjax)项目