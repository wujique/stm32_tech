本目录文件位STM32F103VET教学板相关文件

本目录发布与百度云与github，

* 百度云地址：https://pan.baidu.com/s/12o0Vh4Tv4z_O8qh49JwLjg，目录W108_F103_Tech

* github:

* gitee镜像：

* 文档同步发布在：

#### 目录说明

本工程包含代码、资料、文档，目录说明如下。

| 目录 | 说明                                                         |      |
| ---- | ------------------------------------------------------------ | ---- |
| code | 例程代码，部分教程无代码                                     |      |
| data | 硬件资料：<br />HDK是硬件原理图和丝印说明。<br />REF是主控相关资料。<br />SDK为ST提供的相关库。<br />开发板产品说明书 |      |
| doc  | 文档，本教程文档使用sphinx+markdown编写。源文档在source目录。使用浏览器打开build\html目录中的index.html，可用网页方式查看文档。 |      |
| ref  | 教程相关的文档和工具                                         |      |

#### 文档说明

本教程文档使用markdown格式，使用Typora编写。

所有文档使用Sphinx管理。

doc\source目录下的md文件为文档源文件。包含Advanced、base、spec等子目录的文件。

md文件经过sphinx处理后，生成html文件。文件入口在doc\build\html中的index.html，使用浏览器打开即可浏览所有文档。

sphinx可以将文档处理为pdf，如有需要，可自行转换。