# 在10.10.7.237(Centos 7)上的安装过程：

## 安装编译工具
- yum groupinstall "Development tools"
- yum install git autoconf automake autogen libpcap-devel libtool

## 下载源码
- git clone https://github.com/ntop/nDPI.git

## 编译
- ./autogen.sh
- ./configure
- make && make install

## 测试
### 在em1接口监听30s
- ndpiReader -i em1 -s 30

### 添加简单的协议识别
- echo 'host:"ntop.org"@nTop'> protos.txt
- ndpiReader -i em1 -p protos.txt -s 30

## 维护
本次安装在/root/ndpi/nDPI

## Misc
- 添加json-c支持 即最后打印的统计结果输出到json文件 
  - yum install json-c json-c-devel

## 其他安装
- https://github.com/hackeryard/nDPI/blob/dev/README.nDPI
