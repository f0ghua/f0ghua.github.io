<!--
.. title: 使用Nikola创建个人博客
.. slug: shi-yong-nikolachuang-jian-ge-ren-bo-ke
.. date: 2021-02-22 20:12:13 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

## Windows
### 安装
我Windows机器上用的是winpython3.8.7，按照Nikola的官方文档`Getting Started`，在winpython的命令窗口中输入以下命令进行安装：
```bash
# path D:\xPortable\Tools\python\WPy32-3870\scripts>
> python -m venv nikola
> cd nikola
> Scripts\activate
> Scripts\python -m pip install -U pip setuptools wheel
> Scripts\python -m pip install -U "Nikola"
...snip...
Successfully installed Nikola
```
需要注意的是venv的目录必须在scripts下面创建，不能生成在别处，否则venv的路径会不对。

### github配置
为了保存在github上，直接创建一个github的仓库，名字就是`<github用户名>.github.io`，比如我的就是`f0ghua.github.io`。注意必须按照这个规则来命名，否则将无法访问。
然后在本地创建目录`f0ghua.github.io`，
```
cd f0ghua.github.io
git init
git remote add origin https://github.com/f0ghua/f0ghua.github.io.git
```
按照nikola的官方说明，默认的nikola配置文件会把源码保存在branch src上，depoly的输出保存在master分支。因此，我们创建src分支，来存放源码。
```
git checkout -b src
```
通过nikola命令生成基本的框架，然后复制到f0ghua.github.io目录中去。
```
nikola init mysite
```
添加.gitignore文件以忽略deploy过程中生成的文件
```
cache
.doit.db
__pycache__
output
```
最后，在f0ghua.github.io目录执行以下命令生成静态网页发布到master分支
```
nikola github_deploy
```
这样，访问`f0ghua.github.io`就能看到blog页面了。

## Reference
- [Nikola Getting Started](https://getnikola.com/getting-started.html#)
- [The Nikola Handbook - Deploying to GitHub](https://getnikola.com/handbook.html#deployment)
- [Automating Nikola rebuilds with Travis CI](https://getnikola.com/blog/automating-nikola-rebuilds-with-travis-ci.html)
