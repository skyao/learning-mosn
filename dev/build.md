# build MOSN

## 获取代码

```bash
# 进入GOPATH下的scr目录
cd $GOPATH/src
# 创建 github.com/alipay 目录
mkdir github.com
cd github.com
mkdir alipay
cd alipay
# clone mosn代码
git clone git@github.com:alipay/sofa-mosn.git
mv sofa-mesh sofamesh
cd sofamesh
```

## build

```bash
dep ensure

make build
```

