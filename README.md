# blog

由于刚换了 Mac，为了~~炫耀一下自己的新 Mac~~方便数据同步所以将博客原文也 po 到了 github 上。

## 使用说明

首先要安装 [hexo](https://github.com/hexojs/hexo)，由于 `package.json` 中已经写了，因此只需要执行

```
$ npm install
```

即可。但是在使用时有时会有问题，如[这个 issue ](https://github.com/trentm/node-bunyan/issues/216)说的，因此如果在执行 hexo 命令的时候一直报一些迷之错误，只需要在安装的时候添加 `--no-optional` 参数即可：

```
$ npm install hexo --no-optional
```

安装完 hexo 之后还需要安装主题 [alex](https://github.com/ppoffice/hexo-theme-alex)，安装方式为：

```
$ git clone git://github.com/ppoffice/alex-hexo-theme.git themes/alex
```

这之后最重要的一步就是将配置项写好，由于写起来实在太麻烦，因此将配置项放到了 `./themes` 中，因此在安装完成之后需要执行下面的命令：

```
$ cp themes/_config.yml themes/alex/_config.yml
```

就酱。
