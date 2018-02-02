## Environment

### 概念

> * Environment接口代表了当前应用的运行环境。而应用环境主要包含profiles和properties两部分。
> * profile代表一组需要被容器注入的bean定义，而Environment在其中扮演了决定激活哪些profile的角色。
> * properties即大多数应用中使用的属性配置信息，Environment提供了一种便利的方式从多中属性配置渠道获得属性值。

### PropertyResolver

PropertyResolver接口定义了通过指定key查找对应value和替换字符串中通配符($(...))的方法，起到了属性解决的作用。而Environment接口则继承了PropertyResolver接口，并增加了profile功能。

Environment中的profile相关方法包括：1.获取激活的profile集合 2.获取默认的profile集合 3.检查当前Environment是否支持指定profile集合（激活/默认）。这里需要注意的是第三个方法，如果传入的profile带!前缀，则代表判断反转。

