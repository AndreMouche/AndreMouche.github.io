---
layout: page
title: "About"
description: "Hey, this is Shirly Wu."
---

## 个人简介

<p>Hi,我叫<code>Shirly</code>.</p>

<p>我呢，崇尚自由，热爱冒险，喜欢剥削，迷恋topcoder,曾因为对黑客男神的极度仰慕崇拜而学习编程哈。</p>

<p>作为一个Acmer败类，脑子有点直，不太会说话，做事只知道input和output。 工作不少年，技术没长进，只有对技术和技术男热爱的心依旧哈。</p>

<p>虽然又笨又懒，但想做大牛的心六年如一日，为给自己的青春留下些美好的回忆，故开此博客，企图一入github深似海，从此独立又自强（原本想说从此男神成邻居，低调低调哈）。</p>

<p>虽说人生若只如初见，与技术相遇近六年，迷恋依旧，期待与志同道合的童鞋一起进步哈～</p>


## 成长足迹

* [github首页](http://github.com/AndreMouche)
* [年轻时的博客](http://www.cnblogs.com/AndreMouche/)
* [年少时的博客](http://blog.sina.com.cn/myacmsky)
* [伪文艺时期的豆瓣](http://www.douban.com/people/AndreMouche/)

## 与我联系

* [邮件](mailto:AXuelianWu@gmail.com)
* [新浪微博](http://weibo.com/happyprogrammer)


## 友情链接

* [Liquid 模板引擎](https://github.com/Shopify/liquid/wiki/Liquid-for-Designers)
* [Jekyll 静态博客生成器](http://jekyllrb.com/)




{% if site.disqus_username %}
<!-- disqus 评论框 start -->
<div class="comment">
    <div id="disqus_thread" class="disqus-thread">

    </div>
</div>
<!-- disqus 评论框 end -->

<!-- disqus 公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = "{{site.disqus_username}}";
    var disqus_identifier = "{{site.disqus_username}}/{{page.url}}";
    var disqus_url = "{{site.url}}{{page.url}}";

    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<!-- disqus 公共JS代码 end -->
{% endif %}
