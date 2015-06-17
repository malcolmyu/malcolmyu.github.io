# blog

为了方便数据同步所以讲博客原文也 po 到了 github 上。

## 使用说明

首先要安装 [hexo](https://github.com/hexojs/hexo)，由于 `package.json` 中已经写了，因此只需要执行

```
$ npm install
```

即可。安装完 hexo 之后还需要安装主题 [alex](https://github.com/ppoffice/hexo-theme-alex)，安装方式为：

```
$ git clone git://github.com/ppoffice/alex-hexo-theme.git themes/alex
```

这之后最重要的一步就是将配置项写好，由于写起来实在太麻烦，因此将配置项放到了 `./themes` 中，因此在安装完成之后需要执行下面的命令：

```
$ cp themes/_config.yml themes/alex/_config.yml
```

就酱。
