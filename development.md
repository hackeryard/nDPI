# nDPI开发文档

## 安装

- https://github.com/hackeryard/nDPI/blob/dev/Centos_install.md

## 源代码

- 代码位置：服务器 10.10.7.237: /root/ndpi/nDPI
- 为了修改测试方便，使用部署在10.10.7.232上的git服务，开发时首先要克隆此仓库，使用命令：

  - git clone git@10.10.7.232:~/repo/nDPI.git
- ndpiReader.c是官方给的示例程序，在example/下，通过阅读和调试此程序，得到nDPI的逻辑
- 由于pvs1机器无法访问内网git server，所以需要在237上通过scp把打包的文件上传过来后重新编译，注意不能直接把237机器上编译好的ndpiReader直接scp到pvs1，因为pvs1为ubuntu，与237的centos上某些依赖库的版本不同，并不兼容

## 开发环境

- 单独测试，也即不使用DPDK的流量，在内网的237服务器上测试：连接内网->克隆此仓库->修改源代码->在git shell中执行./re.sh（example/），输入密码，上传更新到232服务器->在237服务器上./remake.sh（/root/ndpi/)，确认编译没有错误，有错误则重复以前的操作->执行ndpiReader -i test.pcap -F flow_file_name > debug.txt，从文件读入，并写结果到当前目录下的flow_file_name；或者执行ndpiReader -i em1 -s 20 -F flow_file_name> debug.txt，从本地网卡em1读入，持续监听20秒，并写结果到flow_file_name；或者解注释udp接收的代码，使用udp实时接收来自DPDK的流量并分析（在pvs1机器上）

## 阅读建议

- 先看一遍程序逻辑
- 接着阅读和调试代码
- 遇到问题再来看一下

## nDPI程序逻辑

- 说明：在程序中，所有的//注释都是后来加的，对理解程序逻辑有帮助，原来的注释方式为/* ... */，另外为了说明简单，只分析单线程的处理逻辑

- main->test_lib->openPcapFileOrDevice ->setupDetection->创建包处理线程，处理函数为processing_thread -> runPcapLoop->pcap_loop()不断接受来自网卡的流量（如果要实时利用DPDK采集的流量进行分析，需要采用一种进程间通信方式，本次采用UDP的方式接收来自DPDK的数据包，即每个UDP报文承载一个数据包，不使用TCP的原因是TCP存在粘包问题） ->pcap_process_packet，实际的包处理函数，参数为header和packet原始数据->ndpi_workflow_process_packet ,完成实际的协议检测工作，返回值为ndpi_proto 结构体，其中app_protocol为顶层应用，比如youtube、qq之类，为nDPI特有，master_protocol为一般需要识别的协议，比如DNS、SSL、HTTP等-> printResults 打印聚合结果，聚合的具体数据，包括所有IP数据的大小，IP数据包的数量等，这些东西存储在ndpi_thread_info[thread_id].workflow中  ->pcap_close关闭pcap_handler->terminateDetection()清理线程申请的其余空间-> runker_packet_store，自己添加的函数，用于存储所有flow的所有包到文件中
- ndpi_workflow_process_packet 逐层解析数据包，解析出vlan_id, iph, iph6, ip_offset等信息，呈递给packet_processing 继续处理，即 使用get_ndpi_flow_info函数，首先使用五元组信息的累加值，除以num_roots，定位到某棵二叉树，然后根据ndpi_workflow_node_cmp的比较规则，通过ndpi_tfind 来查找二叉树中是否存在此flow的节点，如果存在，直接返回，否则在此二叉树中添加一个flow节点，然后返回此节点结构体，类型为ndpi_flow_info
- 在packet_processing 中，新增在flow中存储每个数据包的功能，因为在单个flow中这些数据包都需要存储，而原始的程序的处理方式是前一个包覆盖下一个包，导致并不是所有的包都存储在flow中，所以新增链表结构体来存储所有的包，增加的结构体为ndpi_flow_struct->ndpi_flow_info->raw_packet_struct，存储了每次接收到的单个数据包的信息，包括长度、时间戳，指向下一个数据包的指针
- 在所有数据包处理完成之后，在terminateDetection中的函数清理所有申请的空间之前，需要通过遍历的方式找出所有flow下的所有数据包，具体函数为runker_packet_store，使用递归的方式后序遍历num_roots个二叉树，并遍历其中raw_packet_struct的链表，并定义flow和packets存储的格式，最后使用fwrite依次写入这些数据
- flow和packets存储的格式为：
   - hashval，也即flow_ID
  - number of packets，即数据包的数目
  - detected protocol，即检测出的协议
  - packet 0
    - packet_len
    - timestamp
    - packet_raw_data
- 重要结构体解析：
  - ndpi_flow_info 
    - hashval 五元组hash值
    - src_ip dst_ip 源目的ip
    - src_port dst_port 源目的端口
    - vlan_id 针对vlan流量
    - detect_completed 协议是否检测完成
    - bidirectional 流量是否为双向
    - *ndpi_flow 指向当前流的指针
    - detected_protocol 检测出的协议放在ndpi_protocol结构体中
  - ndpi_flow_struct
    - packet_counter_all 新增 用于包计数
    - packet_counter 应该是检测到协议的时候的包计数
    - bittorrent_stage等 用于protocols目录下的具体协议检测，扩展其他协议的检测需要编写插件，必要时需要在此结构体中添加成员
    - ndpi_packet_struct 存储当前数据包的数据 但由于是覆盖性的 不使用这个结构
    - *ndpi_flow_struct 指向下一个flow的指针？此处存疑，具体看代码
    - raw_packet_struct 指向第一个数据包的指针，为数据包链表的第一个元素
  - raw_packet_struct 
    - len 数据包长度
    - tv 时间戳
    - data[MTU_SIZE] 开辟MTU大小来存储包原始数据
    - *next_packet 指向下一个raw_packet_struct
  - ndpi_node（使用uthash库作为搜索二叉树的实现）
    - char *key 在本例中是指向node_flow_info
    - *left *right node的左右指针

## flowfile读取和分析

- flow和packets存储的格式已在程序逻辑中给出，读取flowfile的读取程序为read_flow.cpp(/root/ndpi/)，已添加每个flow的平均包长和包长的标准差的计算，可以根据需要提取更多的特征。read_flow程序的使用方法如下：

  - ./read_flow flow_file_name (PS: flow_file_name为flow文件名)
- 标准差的计算使用了c++模板，standard_deviation函数可以接受任意类型的数组作为参数，如len为int，timestamp为float等
- 根据需要的方式将特征和协议类型组成的训练行输出到文件

## debug

- [内存泄露](http://scottmcpeak.com/memory-errors/)
- gdb+coredump调试

## 当前工作

- 在pvs1机器的/home/pvs1/pcap_file上分流all_in_one_file_0_1.pcap等四个文件到flowfile_0_1，这些文件可以通过read_flow程序(/home/pvs1/pcap_file/)读取并分析flow文件
- 备注：每个all_in_one_file_0_1.pcap大小为2.5G，分流完成需要半个小时，为了节省分析每个pcap文件的时间，可以同时执行多个ndpiReader，使得分析完最多32个（等于核心数）2.5G的pcap文件的总时间为半个小时：
  - nohup ndpiReader -i all_in_one_file_0_1.pcap -F flowfile_0_1 &
  - nohup ndpiReader -i all_in_one_file_1_1.pcap -F flowfile_1_1 &
  - ......
