---
layout: post
title:  用Jsoup写爬虫
date:   2015-12-22
---

<p class="intro"><span class="dropcap">J</span>soup 是一款Java 的HTML解析器，可直接解析某个URL地址、HTML文本内容。它提供了一套非常省力的API，可通过DOM，CSS以及类似于jQuery的操作方法来取出和操作数据。利用这些特性，我们可以用来爬取各个网站，获得需要的数据。

我利用空余时间写了一个爬取几个内容网站的数据到自己的页面。由于没有采用预先下载然后读取数据库的方式，而是直接爬取交给后台处理然后传给前台，所以效率不是很高，尤其是当爬取的网站过多或者内容过多时，速度明显不如读取数据库的方式。

 下面以我的代码段说明解析过程：

使用之前需要引入jsoup.jar包，jsoup官网[http://jsoup.org][2]。
解析方法 extract具体实现：
{% highlight java %}
//在外面对每一个url（Rule）做一个初始化，包含获取连接的大概数量size等
`

```
public static List<LinkData> extract(Rule rule, int size) {
    List<LinkData> datas = new ArrayList<LinkData>();
    LinkData data = null;
    try {
        //获得url和过滤条件和过滤类型，如class=“xxx”关键字
        String url = rule.getUrl();
        String elementFilter = rule.getElementFilter();
        String type = rule.getType();
        //post或者get
        int requestType = rule.getRequestMoethod();
        //解析第一步，连接url
        Connection conn = Jsoup.connect(url);
        // 设置请求类型
        Document doc = null;
        //获得整个document
        switch (requestType) {
            case Rule.GET:
                doc = conn.timeout(3000).get();
                break;
            case Rule.POST:
                doc = conn.timeout(3000).post();
                break;
        }

        //处理返回数据
        Elements results = new Elements();
        //这里用switch比较好，因为SAE的JDK版本限制，JDK1.6只支持switch（int），所以我用if重写了这段
        if (type == "class") {
            results = doc.getElementsByClass(elementFilter);
        } else if (type == "id") {
            Element result = doc.getElementById(elementFilter);
            results.add(result);
        } else if (type == "tag") {
            results = doc.getElementsByTag(elementFilter);
        } else {
            //当elementFilter为空时默认去body标签
            if (TextUtil.isEmpty(elementFilter)) {
                results = doc.getElementsByTag("body");
            }
        }
        //对不同大小的返回结果作处理，适当删减内容，保证效率
        if (size > 7) {
            size = results.size();
        }
        for (int i = 0; i < size; i++) {

            //选择a标签的链接
            Elements links = results.get(i).select("ahref");
            for (Element link : links) {
                //获取标题和url
                String linkText = link.text();
                String linkHref;
                if (rule.isPreHref()) {
                    linkHref = rule.getRootUrl() + link.attr("href");
                } else {
                    linkHref = link.attr("href");
                }
                
                data = new LinkTypeData();
                data.setLinkHref(linkHref);
                data.setLinkText(linkText);
                data.setSummary(rule.getTag());
                data.setContent(rule.getFrom());
                datas.add(data);
            }

        }

    } catch (IOException e) {
        e.printStackTrace();
    }
    return datas;
}
```

`

{% endhighlight %}

[1]:	http://tags.danlvse.com
[2]:	http://jsoup.org
