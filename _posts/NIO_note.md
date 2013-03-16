#Java NIO 笔记


Java New IO(NIO)是JDK1.4中对原IO api的改进。主要改进点包括：

1. 为原始类型数据提供buffer
2. 字符集编解码
3. 支持基于Perl风格正则匹配
4. 提供一个新的原始IO抽象，Channel
5. 提供支持锁和内存映射的接口
6. 为编写可伸缩的服务器提供多路复用（multiplexed）、非阻塞（non-blocking）IO

几个概念：

1. Buffers：存放数据的容器（类似于数组的线型表结构），可以被Channel直接读写
2. Charsets&decoder&encoder：用于将数据进字符转换
3. Channels：表示一个与实际的IO设备的连接
4. Selectors： 表示有有IO event的channel集合
5. SelectionKeys：维护IO event状态和绑定


