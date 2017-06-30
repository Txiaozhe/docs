#### RNPolymerPo

* [GitHub](https://github.com/yanbober/RNPolymerPo)

RNPolymerPo 是一个基于react-native的生活类聚合实战项目，目前只有android版本。



![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/home_page.png)

![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/weixin_page.png)

![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/mine_page.png)

![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/online_news_page.png)

![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/chart_page.png)

![](https://github.com/yanbober/RNPolymerPo/raw/master/doc/movie_page.png)

* 数据支持（提供黄历、新闻等数据）
  * [聚合数据](https://www.juhe.cn/)
  * [天行数据](http://www.tianapi.com/)


* 获取及运行步骤

```shell
 $ git clone https://github.com/yanbober/RNPolymerPo.git
 $ cd RNPolymerPo
 $ npm install 
```

* 为了便于运行，可以把签名的代码注释


```javascript
...
signingConfigs {
    release {
        //storeFile file(MYAPP_RELEASE_STORE_FILE)
        //storePassword MYAPP_RELEASE_STORE_PASSWORD
        //keyAlias MYAPP_RELEASE_KEY_ALIAS
        //keyPassword MYAPP_RELEASE_KEY_PASSWORD
    }
}
...
```

* 运行

```shell
$ react-native run-android
```

