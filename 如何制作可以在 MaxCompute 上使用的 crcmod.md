# 如何制作可以在 MaxCompute 上使用的 crcmod

之前我们介绍过在 PyODPS DataFrame 中使用三方包。对于二进制包而言，MaxCompute 要求使用包名包含 cp27-cp27m 的 Wheel 包。但对于部分长时间未更新的包，例如 oss2 依赖的 crcmod，PyPI 并未提供 Wheel 包，因而需要自行打包。本文介绍了如何使用 quay.io/pypa/manylinux1_x86_64 镜像制作可在 MaxCompute 上使用的 Wheel 包。

本文参考 https://github.com/pypa/manylinux ，quay.io/pypa/manylinux1_x86_64 镜像也是目前绝大多数 Python 项目在 Travis CI 上打包的标准工具，如有进一步的问题可研究该项目。

准备依赖项
不少包都有依赖项，例如 devel rpm 包或者其他 Python 包，在打包前需要了解该包的依赖，通常可以在 Github 中找到安装或者打包的相关信息。对于 crcmod，除 gcc 外不再有别的依赖，因而此步可略去。

修改 setup.py 并验证（建议在 Mac OS 或者 Linux 下）
较旧的 Python 包通常不支持制作 Wheel 包。具体表现为在使用 python setup.py bdist_wheel 打包时报错。如果需要制作 Wheel 包，需要修改 setup.py 以支持 Wheel 包的制作。对于一部分包，可以简单地将 distutils 中的 setup 函数替换为 setuptools 中的 setup 函数。而对于部分自定义操作较多的 setup.py，需要详细分析打包过程，这一项工作可能会很复杂，本文就不讨论了。

例如，对于 crcmod，修改 setup.py 中的

```js
from distutils.core import setup
```

为

```js
from setuptools import setup
```

即可。

修改完成后，在项目根目录执行

```js
python setup.py bdist_wheel
```

如果没有报错且生成的 Wheel 包可在本地使用，说明 setup.py 已可以使用。

准备打包脚本
在项目中新建 bin 目录，并在其中创建 build-wheel.sh：

```js
mkdir bin && vim bin/build-wheel.sh
```

在其中填入以下内容：

```js
#!/bin/bash
# modified from https://github.com/pypa/python-manylinux-demo/blob/master/travis/build-wheels.sh
set -e -x
 
# Install a system package required by our library
# 将这里修改为安装依赖项的命令

# Compile wheels
PYBIN=/opt/python/cp27-cp27m/bin
# 如果包根目录下有 dev-requirements.txt，取消下面的注释
# "${PYBIN}/pip" install -r /io/dev-requirements.txt
"${PYBIN}/pip" wheel /io/ -w wheelhouse/
 
# Bundle external shared libraries into the wheels
for whl in wheelhouse/*.whl; do
    auditwheel repair "$whl" -w /io/wheelhouse/
done
```

将第一步获知的依赖项安装脚本填入此脚本，在使用 python 或 pip 时，注意使用 /opt/python/cp27-cp27m/bin 中的版本。

最后，设置执行权限

```js
chmod a+x bin/build-wheel.sh
```

打包
使用 Docker 下载所需的镜像（本步需要使用 Docker，请提前安装），此后在项目根目录下打包：

```js
docker pull quay.io/pypa/manylinux1_x86_64
docker run --rm -v `pwd`:/io quay.io/pypa/manylinux1_x86_64 /io/bin/build-wheel.sh
```

完成的 Wheel 包位于项目根目录下的 wheelhouse 目录下。
