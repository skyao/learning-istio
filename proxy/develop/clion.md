# 使用CLion

为了体验的一致，IDE选择了CLion，这样Java用IntelliJ IDEA，Golang用Goland，c++用CLion，全家桶。

## Linux下

在CLion中导入项目，开始有大量错误，后来在 make build 通过之后，就一切正常了。

建议在导入项目之前，就先通过 make build 命令先完成构建。

## Mac下

同样的方式，先导入项目，大量出错，然后 make build 通过之后，mac下的CLion不知为何没有正常工作，依然是大量错误。

尝试删除项目，删除本地clone下来的proxy代码，重头开始先 make build，再导入项目到CLion中。

TBD：稍后搞定再更新。