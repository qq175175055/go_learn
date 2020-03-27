
# go安装

go安装包下载：https://studygolang.com/dl

根据操作系统下载相应的文件，然后安装即可

# 国内无法使用go get的问题解决

国内因为代理的问题，通常无法使用go get。以windows为例，可以采用如下简单办法解决：

1、打开git bash

2、输入export GOPROXY=https://goproxy.io

3、再使用go get，例如go get -u github.com/lukehoban/go-outline

就ok了

下载的源码和文件会在GOPATH的目录下，可以在git bash中使用go env查看GOPATH的目录
