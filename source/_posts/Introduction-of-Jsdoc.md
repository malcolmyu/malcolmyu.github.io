title: JSDoc 配置使用概览
date: 2015-04-25
categories: 开发工具
tags: [jsdoc]
toc: true

---
尽管一个好的项目文档能让项目增光添彩，但文档的作用始终是锦上添花而非雪中送炭，对于开发者来说，费尽心神的整理项目文档似乎也并非是一件令人愉快的事情。在使用题中所述的工具——jsdoc和编写本文的同时，笔者也是几度懊恼：这东西在项目中真的有用吗？是不是有点浪费时间？但随着项目体量的增大和开发人员的增多，使用并自动化维护一份好的文档所带来的优势也是逐渐显现。笔者也决心写一篇科普小文，说一说文档工具的使用注意要点。

<!--more-->

# JSDoc 命令行使用方法

通过命令行来生成 JSDoc 有两种适合前端的方式：一种是通过 npm 来安装 jsdoc 来生成文档，需要学习相关的命令行参数与配置文件的编写；另一种是使用 grunt 来生成 jsdoc，这种方法配置起来比较简单，等于将各种配置参数写在了 Gruntfile.js 的配置项里，然后可以自定义各种操作命令，十分灵活。下面就分别就这两种方法介绍一下。

## 通过 npm 生成

通过 npm 的生成方式主要包含安装、配置和生成三步。安装的方式如下所示：

{% codeblock lang:bash %}
    npm install jsdoc -g
{% endcodeblock %}  

然后去指定文件（如 `test.js`）的路径下运行

{% codeblock lang:bash %}
    jsdoc test.js
{% endcodeblock %}      

文档即可被生成在根目录下的 out 文件夹中。

上面的过程看上去十分简便，但我们省略了配置这一步，如果我们不想生成在 out 文件夹下怎么办？我们想针对一个项目工程生成文档怎么办？觉得原有文档模板太丑想换个风格怎么办（_说句实话个人感觉默认模板是目前见过所有模板中最好看的……_）？这就需要在配置这一步中进行繁杂的个性化指定。对于项目的个性化指定可以通过**配置文件**和**命令行参数**两种方式来设置，先说一下命令行参数的配置。

有关命令行参数的配置可以参考[官网文档](http://usejsdoc.org/about-commandline.html)的详细说明，这里只列出几个重要的参数：

|参数|全称|说明|
|:---|:---|:---|
|`-c`|`--configure`|引入配置项，默认为 jsdoc 安装目录的 `config.json` 文件|
|`-d`|`--destination`|配置文档输出的目录，默认为 `./out`|
|`-P`|`--package`|可以将 `package.json` 文件写入文档中，默认写入当前路径的第一个 `package.json`|
|`-r`|`--recurse`|递归调用路径的子目录查找 js 文件，当生成一个文件夹下的全部 js 的文档时必须使用这个参数|
|`-R`|`--readme`|可以引入一个说明文件，默认将当前路径中的第一个 `readme.md` 文件添加到文档中|
|`-t`|`--template`|可以给文档指定一个第三方的模板|

举个例子，比如我们要将 `./src` 路径下的所有 js 文件生成文档，然后存放在 `./docs` 文件夹中，操作如下：

{% codeblock lang:bash %}
    jsdoc src -r -d docs
{% endcodeblock %}  

但是这样感觉还是不方便，如何对特定文件进行文档解析呢？如果有一些文件不想进行文档解析如何从中间剔除呢？而且每次都要输入如此复杂的命令行参数也十分不方便。要解决这些问题，就需要用到配置文件 `conf.json` 了。

对于配置文件的详细解释可以参见[官方文档](http://usejsdoc.org/about-configuring-jsdoc.html)的说明，这里也是举一个简单的例子：

{% codeblock lang:json %}
{
    "tags": {
        "allowUnknownTags": true
    },
    "source": {
        "include": ["./src/lib", "./src/util"],
        "exclude": ["./src/lib/demo"],
        "includePattern": ".+\\.js$",
        "excludePattern": "(^|\\/|\\\\)_"
    },
    "opts": {
        "recurse": true
    }
}
{% endcodeblock %}

- `tags.allowUnknownTags` 表示允许使用为定义的 tag 标签，不然使用未知标签会报错；
- `source.include` 表示需要进行文档解析的文件（夹）路径；
- `source.exclude` 表示不需要解析的文件（夹）路径；
- `source.includePattern` 表示要进行文档解析的文件类型，支持正则进行文件名匹配，上文表示只能解析 *.js 文件；
- `source.excludePattern` 表示不要进行解析的文件类型，上文表示所有以下划线开头的文件和文件夹都不被解析；
- `opts.recurse` 表示是否递归解析文件夹，上文表示继续拧递归解析，即使用 `-r` 参数。

这里的 `opts` 里面的内容与上表中命令行参数配置的内容一致，即也可以在此处配置各种参数，详情官网文档中有叙述。将上文保存为一个 json 格式的文件，然后使用 `-c` 参数引入，就可以按照设定好的规则直接批量生成文档了。

{% codeblock lang:bash %}
    jsdoc -c conf.json
{% endcodeblock %}

## 通过 grunt 生成

在项目中如果使用了 grunt 工具，也可以将 jsdoc 集成到 grunt 中去。这种方式十分的简便，在其 [github 地址](https://github.com/krampstudio/grunt-jsdoc)上也有详尽的介绍，具体流程与其他的 grunt 模块差别不大，使用如下命令进行安装：

{% codeblock lang:bash %}
    npm install grunt-jsdoc --save-dev
{% endcodeblock %}

然后在 `Gruntfile.js` 文件中进行配置，就可以在命令行中使用了：

{% codeblock lang:js %}
    grunt.initConfig({
        jsdoc: {
            dist: {
                // 必填项，需要生成文档的路径数组，也可以将 README.md 文件加入其中
                src: ['src/*.js', 'test/*.js'],
                // 可选项，jsdoc bin 文件路径，一般不写，会自己在 node_modules 中寻找
                jsdoc: '',
                options: {
                    // 必填项，生成文件的路径
                    destination: './docs/',
                    // 可选项，conf 文件的路径
                    configure: 'conf.json',
                    // 可选项，模板路径
                    template: ''
                }
            }
        }
    });
    grunt.task.loadNpmTasks('grunt-jsdoc');
    grunt.task.registerTask('doc', ['jsdoc']);
{% endcodeblock %}

{% codeblock lang:bash %}
    grunt doc
{% endcodeblock %}

需要注意的是，配置项中的 `options` 内容与 `conf.json` 中 `opts` 的内容也一致。

# JSDoc 注释规范

上面的一堆其实都是虚的，最主要的还是在代码中按照 JSDoc 要求的注释规范详尽的写清楚注释。JSDoc 的注释有一条很重要的规范就是：

>  It must start with a `/**` sequence in order to be recognized by the JSDoc parser. Comments beginning with `/*`, `/***`, or more than 3 stars will be ignored. This is a feature to allow you to suppress parsing of comment blocks.

> 必须以 `/**` 开头才能被 JSDoc 的解析器识别。 以 `/*`、`/***`、 或者超过三个 `*` 开头的注释我们都不管。这可是我们做的一个 feature 哟~它可以帮助你避免解析你不想解析的注释。

使用 `/**` 开始注释之后，就需要研究一下注释的标签（Tags），JSDoc 提供了几十种标签，满足我们各种稀奇古怪的需求。我们先从单个文件的注释说起，对 JSDoc 的注释规范进行说明。

## 单个文件注释

提到单个文件注释，首先要说到 JSDoc 里面的一个概念，换做**块级标签（Block Tag）**与**行内标签（Block Tag）**，整的这么玄乎，无非就是说有些标签是用 `@` 打头起一行，叫做块级；有些标签是用 `{}` 包裹放在行内，最常见的就是 `@type` 和 `@link` 标签，可以放在行内对变量进行类型说明与解释。如：

{% codeblock lang:js %}
    /**
     * 大哥大嫂过年好
     * @param {string} parent - 你是我的爷
     * @param {string} child - 我是你的儿
     */
    function happyNewYear(parent, child) {
        // ...
    }
{% endcodeblock %}

以下是单文件注释中常用的标签：

|标签|说明|
|:---|:---|
|`@alias`|同名引用，用于指定一个同名属性或在非显示的情况下标明从属关系，详见下节|
|`@author`|说明这篇代码谁写的，~~方便出 bug 的时候削人~~|
|`@class` `@constructor`|标记一个函数为构造函数，可以使用 `new` 来实例化|
|`@constant` `@const`|将一个变量标记为常量|
|`@description` `@desc`|进行描述，一般会把注释开头的文字默认作为描述|
|`@enum`|标注一个对象为枚举对象|
|`@example`|可以给文档提供一个如何使用的例子|
|`@file` `@fileoverview` `@overview`|表示对一个文件的描述|
|`@global`|标记一个全局变量|
|`@param`|标记一个函数的参数|
|`@returns` `@return`|标记一个函数的返回值|
|`@this`|标注一个 `this` 关键字的指向|

但是在项目中，我们可能拥有一堆又一堆的单个文件都需要添加注释，总不能把这些方法啊变量啊的注释都丢到一个文档页面中吧。不用担心，JSDoc 提供了一种模块化注释的方式帮我们解决这一问题。

## 模块化注释

JSDoc 中对模块的表示共有三种方法，分别为**类（`@class`）**、**模块（`@module`）**和**命名空间（`@namespace`）**。使用上面三个标签进行标注的话，其从属的属性都可以使用长名进行引用。`@module` 顾名思义是用来标注一个 JS 模块的，一般用这个模块再被 `require` 的时候的写法作为名字标注。比如这样一个模块：

{% codeblock lang:js %}
    /**
     * 表单校验模块
     * @module lib/validator
     */
    
    /** 校验通过方法 */
    exports.valid = function() {};
    /** 校验失败方法 */
    exports.inValid = function() {};

    // 另一个文件在使用时需要这样用
    require('lib/validator');
{% endcodeblock %}

正如上文所写，对于一个模块需要 `exports` 的方法可以直接写双星注释，会被 JSDoc 所自动识别。但有些情况下为了我们可能会先给一个本地对象挂一堆方法然后把这个对象直接 `exports`。为了标明它们的从属关系，我们就可以用到 `@alias` 别名标签。如下例：

{% codeblock lang:js %}
    /**
     * 表单校验模块
     * @module lib/validator
     */
    
    /**
     * @alias module:lib/validator.valid
     */
    var valid = function() {};

    exports.valid = valid;
{% endcodeblock %}

注意到上文中别名标签的值，这里用到一个叫做**命名路径（namepaths）**的概念。这个概念表示使用 `#` `.` `~` 三种符号表示两个模块之间的从属关系，其分别为：

- `#` 实例属性，表示实例对象能够继承的属性；
- `.` 静态属性，普通对象的静态属性；
- `~` 内部属性，在函数对象内部作用域定义的属性。

下面这个例子分别解释了上面三种属性：

{% codeblock lang:js %}
    /** @constructor */
    function Person() {
        this.instanceSay = function() {
            return "我是实例属性";
        }

        function innerSay() {
            return "我是内部属性";
        }
    }
    Person.staticSay = function() {
        return "我是静态属性";
    }
    // 在注释中我们就可以使用这样的方式来表示上面三种属性
    // Person#instanceSay
    // Person.staticSay
    // Person~innerSay
{% endcodeblock %}

上面用命名路径生成的诸如 `Person#instanceSay.hello~hi` 这样的引用名称就叫做**长名（longname）**。其中如果是 `@module` 模块的话，需要添加 `module:` 前缀，用于与命名空间相区分。而命名空间的作用就是给其子属性开启一个可以被挂载的空间，可以在文档中被单独标记。子属性可以使用 `@memberof` 对父属性进行挂载。比如我们挂载到全局变量上的属性，并不遵循模块化的风格，对其进行标记就可以使用命名空间的方式，如：

{% codeblock lang:js %}
    /**
     * 某个文件中的祖先空间
     * @namespace ancestor
     */
    var ancestor = {};
    window.ancestor = ancestor;

    /**
     * 某个文件中的父亲空间，挂载到祖先之下
     * @namespace parent
     * @memberof ancestor
     */
    var parent = {};
    window.ancestor.parent = parent;

    /**
     * 某个文件中的孩子空间，挂载到父亲之下，需要用长名
     * @namespace child
     * @memberof ancestor.parent
     */
    var child = {};
    window.ancestor.parent.child = child;
{% endcodeblock %}

需要注意的是，这种模块化的标签似乎 JSDoc 都会对其进行作用域的识别，因此在注释的时候一定要注意作用域的问题。比如对一个文件整个注释为 `@module`，其内部的 `@namespace` 可能就会失效，无法在文档上良好的反应。以下是模块化注释的常用标签：

|标签|说明|
|:---|:---|
|`@event`|在模板中标记可以被触发的事件，可与 `@fires` 标签配合使用|
|`@fires` `@emit`|模块通信触发事件描述，需要与 `@event` 标签配合使用|
|`@inner`|模块内部变量标注|
|`@memberof`|标记模块之间的从属关系|
|`@module`|标记一个 CMD 或是 AMD 的模块|
|`@namespace`|开启命名空间|
|`@see`|可以在文档中进行跳转，需要使用长名来进行连接|

_参考链接_

- [jsdoc - Github](https://github.com/jsdoc3/jsdoc)： jsdoc npm module 的 Github 地址
- [jsdoc 官网](http://usejsdoc.org/index.html) ：jsdoc 官方文档

