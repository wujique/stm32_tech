本目录文件为STM32F103VET教学板相关文件

本教程发布于百度云与github，

* 百度云地址：https://pan.baidu.com/s/12o0Vh4Tv4z_O8qh49JwLjg

  目录W108_F103_Tech

* github: <https://github.com/wujique/stm32_tech>

  在github上托管的仅仅是doc目录，其他资料请从百度云下载。

* gitee镜像：

* 文档同步发布在：<https://stm32-tech.readthedocs.io/zh_CN/latest/>

#### 目录说明

本工程包含代码、资料、文档，目录说明如下。

| 目录 | 说明                                                         |      |
| ---- | ------------------------------------------------------------ | ---- |
| code | 例程代码，部分教程无代码                                     |      |
| data | 硬件资料：<br />HDK是硬件原理图和丝印说明。<br />REF是主控相关资料。<br />SDK为ST提供的相关库。<br />开发板产品说明书 |      |
| doc  | 文档，本教程文档使用sphinx+markdown编写。源文档在source目录。使用浏览器打开build\html目录中的index.html，可用网页方式查看文档。<br />文档托管于github |      |
| ref  | 教程相关的文档和工具                                         |      |

#### DOC目录文档说明

`build`是md转换为html的目录。

`source`是md文档目录。其中`spec`是产品说明书，`base`是基础教程，`Advanced`是提高教程。

本教程文档使用**markdown**格式，使用**Typora**编写。

所有文档使用**Sphinx**管理。

md文件经过sphinx处理后，生成html文件，并按照目录结构链接，入口在`doc\build\html`中的`index.html`，使用浏览器打开即可浏览所有文档。

sphinx可以将文档处理为pdf，如有需要，可自行转换。

Typora也可以将md文档转换为pdf。

> 可参考：
>
> 《使用ReadtheDocs托管文档》
>
> <https://www.xncoding.com/2017/01/22/fullstack/readthedoc.html>