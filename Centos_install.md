# ��10.10.7.237(Centos 7)�ϵİ�װ���̣�

## ��װ���빤��
- yum groupinstall "Development tools"
- yum install git autoconf automake autogen libpcap-devel libtool

## ����Դ��
- git clone https://github.com/ntop/nDPI.git

## ����
- ./autogen.sh
- ./configure
- make && make install

## ����
### ��em1�ӿڼ���30s
- ndpiReader -i em1 -s 30

### ��Ӽ򵥵�Э��ʶ��
- echo 'host:"ntop.org"@nTop'> protos.txt
- ndpiReader -i em1 -p protos.txt -s 30