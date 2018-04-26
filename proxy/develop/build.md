# build

## prepare

### bazel

https://docs.bazel.build/versions/master/install-ubuntu.html

```bash
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip python

echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -

sudo apt-get update && sudo apt-get install bazel
```

### cmake

```bash
sudo apt-get install cmake
```

### automake

```bash
apt-get install automake
```

### libtoolize

```bash
buildconf: libtoolize not found.
    You need GNU libtoolize 1.4.2 or newer installed.
```

`https://www.gnu.org/software/libtool/`

```bash
./configure
make
sudo make install
```

## build

```bash
cd proxy
make
```