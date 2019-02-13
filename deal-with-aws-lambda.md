## AWS Lambda 的跳坑指南

目前serverless已经成为一种主流，然而不幸的是，目前的serverless在开发者体验上距离VPS和Docker等环境的差距还是相当大。这里的坑很多，现将笔者踩坑的经验悉数如下。

以AWS Lambda为代表的serverless，其初衷是搭建一套轻量级的，无状态的，并且较少依赖于环境的API接口。其优势在于经济（节省成本）、快捷（不需要浪费太多精力在配置web server、load balancer、log、堡垒机等基础设施上）。

### 一、在使用AWS Lambda时，你会遇到的限制

但是AWS Lambda弊端很明显，即无法搭建自定义的Runtime。AWS官方的Runtime只包括了几门主要语言的基本执行环境，除此以外更无其他。

一般来说，自定义的Runtime包括如下几个方面的定义：

1. 环境变量 environment variables
1. 用户的根目录`~home/`
1. 二进制的包依赖，即用`apt-get` / `yum install`的方式安装的依赖包
    1. 语言环境本身，即Python、Node.js、JDK等
    1. Docker / SSH / Git 等常用工具
    1. 其他的包
1. Python的包依赖，即通过`pip install` 安装的包（或者Node.js的包，或者JAR包等）
1. 程序运行时产生的文件，包括 
    1. 临时文件
    1. log日志文件
    1. 输出结果

如果用Docker或者普通的计算节点，以上都不是问题，但不幸的是，在serverless环境下，上述方面会成为令人头疼的限制：

1. **环境变量 environment variables** —— 在AWS Lambda中可以自行设置
1. **用户的根目录`~home/`** —— 用户根目录的内容 ，既无法通过自定义Layer和Deployment Package来实现静态修改，甚至更无法在Lambda函数执行时动态修改……因为Lambda除了`/tmp`目录可写外，其他目录均只读
1. 二进制的包依赖，即用`apt-get` / `yum install`的方式安装的依赖包
    1. **语言环境本身，即Python、Node.js、JDK等**——AWS官方提供Runtime，但一个Runtime中只能指定一种语言，无法多语言同时运行
    1. **Docker / SSH / Git 等常用工具**——有一些第三方的Layer，比如[这个](https://github.com/lambci/git-lambda-layer)提供了Git+SSH。
    1. **其他的包**——必须通过在开发的工作站上下载[Lambda的AMI](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)，在本地开发环境上搭建AMI的虚拟环境，在其中下载安装了相应的包，将这些包归整打成一个Layer的.zip文件，上传部署到`/opt`环境下……并且体积还有限制
1. **Python的包依赖，即通过`pip install` 安装的python包（或者Node.js的包，或者JAR包等）**——同样无法通过直接在Lambda上跑`pip`，而是需要先在开发环境上先用`pip`把相应的Python包下载下来之后，再按照一定的要求放在.zip文件里，作为Deployment Package的一部分
1. 程序运行时产生的文件，包括 
    1. **临时文件**——只能写在`/tmp`目录下，而不能写在其他文件，包括不能写在Lambda部署时的代码文件同一个相对路径中，因为其是在`/var/task/`目录下
    1. **log日志文件**——只要使用文档中推荐的日志方式，Lambda会代为管理日志，这一点还比较省心
    1. **输出结果（artifacts）**——无法存放在本地，只能通过网络上传到S3或者其他地方

### 二、跳坑指南

除了上述以外，还有很多坑：

#### 关于网速

由于国内上外网速度很慢。所以在AWS中，如果是通过网页方式上传Deployment Package的.zip包，经常会Network Error，这很让人抓狂！解决方案是用AWS提供的CLI命令行工具

#### 关于Python包

Python包首先要放在Deployment Package的.zip包的顶层目录下面，而不是放在.zip包的类似`/venv/Scripts/site-packages`的次级目录下。另外，很多Python包是bdist二进制，导致在本地的环境如果不是和AMI一致就有可能出错的情况。

如果你将Python包放在Layer文件中而不是Deployment Package中，记得要设置`PYTHONPATH`为`/opt/xxx`的环境变量。反正坑已经这么多了，不怕再多加一个：笔者本人也尝试这么做过，但失败了

#### 关于`home`目录

如果想使用Git，有可能会跳出`Could not create directory '/home/sbx_user1070/.ssh'`的提示，解决方案是只要按照[文档里](https://github.com/lambci/git-lambda-layer#complex-example-on-nodejs-w-ssh)说的去做了，就OK了。

### 三、后记
笔者曾经尝试过别的方案。亲测过微软的Azure，感觉还不如AWS Lambda。前者强制在Python 3.6下，后者既支持Python 3.6，也支持3.7，当然还支持Java, Go等语言。另外后者的文档也不如前者。

有人提醒可以用TravisCI, CircleCI, 以及Netlify等工具，看来这是我第一篇写AWS Lambda的，或许是最后一篇。
