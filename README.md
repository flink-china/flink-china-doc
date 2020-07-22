[Flink 中文文档](https://ci.apache.org/projects/flink/flink-docs-master/zh/) 网站是 [Apache Flink 官方文档](https://ci.apache.org/projects/flink/flink-docs-master/) 的中文翻译版。

这篇README主要是介绍如何构建和贡献 Apache Flink 文档/中文文档。

# 环境需求

我们使用 Markdown 来写作，以及 Jekyll 将文档转成静态HTML。使用下面的命令一键安装所需软件：

Markdown处理需要用到 Kramdown，而语义高亮需要用到基于python的 Pygments。如果要用Ruby运行Javascript代码，你需要安装一个javascript引擎（例如 `therubyracer`）。你可以通过下面的命令安装所以需要的软件：

```bash
gem install bundler
bundle install
```

注意在 Ubuntu系统上，有可能需要先通过 apt 安装`ruby-dev`和`python-setuptools`包。

# 使用Docker化的 Jekyll

我们将上述的 jekyll 环境Docker化了。如果你有[docker](https://docs.docker.com/)的话，你可以运行下面的命令来启动容器。

```
cd flink-china-doc/docker
./run.sh
```

第一次构建镜像的过程需要一段时间，不过从第二次开始就可以秒级完成。这个`run.sh`命令会帮你起一个bash会话窗口，在那里你可以运行下面的doc命令。

# 构建

`build_docs.sh`脚本会调用 Jekyll 在`target`目录下生成HTML文档。你可以在浏览器中指定`target/index.html`来阅读。

如果你以预览的方式运行`build_docs.sh -p`，Jekyll会在`localhost:4000`启动一个web服务器，并会自动更新你所做的修改。一般使用这种模式来做翻译/写作。


# 翻译

我们非常欢迎有兴趣的同学参与一起翻译和优化中文文档。欢迎**译者**与**校对**的加入。我们会在[感谢页面](http://doc.flink-china.org/about/#thanks)感谢所有参与了贡献的人。**如有遗漏，请联系我们！**

**如果你想要翻译一篇文章，请先在 [issue](https://github.com/flink-china/flink-china-doc/issues) 页面中查看是否已经有人在翻译这篇文档。如果没有，你可以发布一个 issue 或者回复某个模块的 issue，说明你要认领的文档。**

我们建议你先在本地将英文文档按照原格式翻译成中文，直接在原英文文档基础上编辑翻译，注意保留文档原有的 TOC 、`{% top %}`等内容。并在本地测试验证后，提交你的翻译成果。提交翻译成果有两种方式：

1. 可以在[Flink中文文档](http://doc.flink-china.org)的具体文档页面的最下方找到"在 Github 上编辑此页！"的链接，你可以点击链接，并修改错误或者提交翻译结果。
2. 通过提交[Pull Request](https://help.github.com/articles/using-pull-requests/)的方式提交你的翻译结果。

**注意：**请通过 PR 的方式来提交翻译结果（包括拥有 commit 权限的成员），并至少邀请一位校对人员审核通过后才能被 merge 。在评论中 @flink-china/review  即可邀请所有校对人员。

## 翻译要求

- 汉字，字母，数字等之间以空格隔开。
- 中文使用中文符号，英文使用英文符号。
- 专有词注意大小写，如 Flink，Java，Scala，API，不要用小写的 flink, java, scala, api。
- 术语与已有译文保持一致，如果有不同意见请先在 issue 中讨论。
- 代码只翻译注释，也可不翻译注释。


## 术语翻译对照

- transformations 转换/转换操作
- lazy evaluation 延迟计算 
- exactly-once    只处理一次
- at-least-once   至少处理一次
- at-most-once    最多处理一次
- operator        算子

## 不翻译的术语

DataSet, DataStream, sink, connector。


# 写作指南

关于 Flink 文档的写作指南，请参考 [English README](https://github.com/flink-china/flink-china-doc/blob/master/README_EN.md#contribute)

