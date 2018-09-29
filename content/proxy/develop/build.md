# 构建项目

## 安装Bazel

### Linux下安装Bazel

Linux下安装Bazel可以参考官方安装文档：

https://docs.bazel.build/versions/master/install-ubuntu.html

可以有不同的安装方式，比如通过apt-get命令安装：

```bash
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip python

echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -

sudo apt-get update && sudo apt-get install bazel

## 检查版本
bazel version
```

也可以通过下载二进制包安装：

```bash
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip python

## 下载 bazel-0.14.0-installer-linux-x86_64.sh
chmod +x bazel-0.14.0-installer-linux-x86_64.sh
./bazel-0.14.0-installer-linux-x86_64.sh --user

## 修改~/.bashrc文件，增加内容
export PATH="$PATH:$HOME/bin"

## 重新source
source ~/.bashrc
## 检查版本
bazel version
```

### mac下安装Bazel

直接通过brew命令安装：

```bash
brew install bazel
```

## 安装Bazel相关的类库

为了build proxy模块，需要安装一些和bazel相关的类库，考虑envoy的文档：

https://github.com/envoyproxy/envoy/blob/master/bazel/README.md

linux下：

```bash
 apt-get install libtool
 apt-get install cmake # 这个有问题！
 apt-get install realpath
 apt-get install clang-format-5.0
 apt-get install automake
```

注意不能直接通过apt-get命令来安装cmake，因为版本太久（3.5），需要自行安装新版本。

在 [cmake的官方](https://cmake.org/download/) 下载最新版本，然后自行编译安装：

```bash
wget https://cmake.org/files/v3.11/cmake-3.11.3.tar.gz
gunzip cmake-3.11.3.tar.gz
tar xvf cmake-3.11.3.tar 
cd cmake-3.11.3/
./configure 
make
sudo make install
```

mac下：

```bash
brew install coreutils # for realpath
brew install wget
brew install cmake
brew install libtool
brew install go
brew install bazel
brew install automake
```

## 环境和依赖

> 备注：不是很确认是否真的需要下面的内容

https://www.envoyproxy.io/docs/envoy/latest/install/building.html#requirements

- GCC 5+ (for C++14 support).

- These [pre-built](https://github.com/envoyproxy/envoy/blob/master//ci/build_container/build_recipes) third party dependencies.

  这些依赖，可以在本地envoy目录中找到对应的sh文件，然后`sudo sh **.sh` 进行安装即可。在mac下，有些类库通过brew命令安装会更简单

- These [Bazel native](https://github.com/envoyproxy/envoy/blob/master/bazel/repository_locations.bzl) dependencies.

  这些依赖在make时应该会自动下载(?)。

## 准备好科学上网

构建过程中，要从网上拉取大量的依赖包，因此可能被墙，也可以网络速度非常慢导致下载时间超长。

因此有必要考虑科学上网。

构建过程中，可以通过http_proxy等环境变量将代理服务器传递过去，不过有时又不好使，安全期间，都设置上。

新建`~/.proxyrc`文件，内容如下：

```bash
export http_proxy=http://localhost:8123
export HTTP_PROXY=http://localhost:8123
export https_proxy=http://localhost:8123
export HTTPS_PROXY=http://localhost:8123
export all_proxy=socks5://localhost:11080
export ALL_PROXY=socks5://localhost:11080
export no_proxy='127.0.0.1, localhost, 192.168.1.0/24'
```

在需要翻墙时，就通过source命令载入这些环境变量，然后用curl命令检验：

```bash
source ~/.proxyrc
curl ip.gs
```

## 开始构建项目

```bash
cd proxy
make
```
