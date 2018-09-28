# 理解OutOfMemoryError异常

当在Java堆中不够为对象分配空间时就会跑出`java.lang.OutOfMemoryError`错误.

通常`java.lang.OutOfMemoryError`异常表明内存溢出.有如下几种情况:
  * 当GC无法为新对象分配空间,并且无法继续扩展空间时 ...... 通常时这种情况;
  * 当本地内存不足无法加载java class时 ...... 有时会这样;
  * 当GC执行了很长的时间但是只有一点点内存被释放时 ...... 这种情况极少发生;
  * 当系统内存不足(比如:交换空间过低)时本地方法`native library code`也会抛出这个异常

抛出`java.lang.OutOfMemoryError`时 **stack trace** 会被打印出来.

发生OutOfMemoryError时我们需要找到异常的原因. 是Java堆满了,或者系统内存堆满了?
在异常信息的末尾会有些信息帮助我们找到原因, 比如:
  * **Exception in thread thread_name: java.lang.OutOfMemoryError: Java heap space**   
    原因: **Java heap space**信息表明堆无法为新对象分配空间, 这个异常通常情况下不说明内存溢出,
    而是配置的问题, 指定的堆大小或者时默认的大小无法满足应用程序.  
    另一种情况是, 在程序运行很长一段时间的情况下,这个异常表明程序中有很多对象无法被GC回收,就是Java
    语言中说的内存溢出.   
    还有一种可能时, 程序大量使用了`finalizers`. 一个类如果实现了**finalize**方法, 那么这个类
    的所有对象在GC回收时不会立即释放,而是被放到一个队列中,在Oracle Sun实现的JVM中由一个守护线程
    执行finalize方法,当执行的速度赶不上队列存放的速度时会导致堆满了,从而引发内存不足.通常情况下是
    因为程序创建了优先级比较高的线程,导致守护线程执行的频率降低了,从而执行速度降低了.  

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: GC Overhead limit exceeded**  
    原因: **GC Overhead limit exceeded**表明GC一直在执行,Java程序运行非常缓慢.    
    在GC执行后,如果程序运行时间有98%的时间花费在垃圾回收上,并且只有不到2%的堆空间被释放,这样种情况
    连续出现5次后抛出异常`java.lang.OutOfMemoryError`.这种情况通常时在堆空间只有少量的情况下
    有大量的新对象被创建.    
    办法: 增加堆大小, 或者使用参数`-XX:-UseGCOverheadLimit`关闭这个异常.   

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: Requested array size exceeds VM limit**   
    原因: **Requested array size exceeds VM limit**表明程序或者程序使用的API尝试申请的数组长度超过了
    堆空间大小, 比如: 程序申请512M大小的数组,但是堆大小上限是256M时就会出现这个异常.    
    办法: 通常这种情况是配置问题,配置的空间太少; 或者是程序的BUG.       

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: Metaspace**
    原因: java class元信息(JVM内部用来表示Java class的)被分配在系统内存中(这里用元信息`metaspace`表示).      
    如果类的元信息过大,就会出现Metaspace的OutOfMemoryError异常.参数`MaxMetaSpaceSize`用来控制元信息可以使用的
    空间的上限.当需要的内存大小超过`MaxMetaSpaceSize`就会抛OutOfMemoryError异常.      
    办法: 如果设置了`MaxMetaSpaceSize`可以增大值. MetaSpace和Heap分配来自于相同的地址空间, 减少Heap空间大小会
    增加MetaSpace空间.     

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: request size bytes for reason. Out of swap space?**    
    原因: 通常在VM向系统申请空间失败时并且系统内存使用接近上限时出现的异常.   
    办法: 出现这个异常时,VM会执行错误处理机制,通常会生成一个错误日志文件,里面包含了线程,进程和系统崩溃时间等信息.    

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: Compressed class space**    
    原因: 在64-bit机器上, 使用`UseCompressedOops`可以用32-bit的指针表示class元信息地址.可以通过参数`UseCompressedClassPointers`
    控制(默认使用),如果使用了`UseCompressedClassPointers`class元信息的空间大小就是`CompressedClassSpaceSize`指定的固定大小.
    如果`UseCompressedClassPointers`使用的空间超过了`CompressedClassSpaceSize`就会抛**Compressed class space**的OutOfMemoryError异常.   
    办法: 增加`CompressedClassSpaceSize`大小可以关闭`UseCompressedClassPointers`.     
    > 注意: `CompressedClassSpaceSize`大小是有限制的, 如: -XX: CompressedClassSpaceSize=4g 超过了可接受的限制就会抛出类似于:_CompressedClassSpaceSize大小4294967296无效,必须在1048576到3221225472之间_ 的消息.   

  * **Exception in thread thread_name: java.lang.OutOfMemoryError: reason stack_trace_with_native_method**   
    原因: 如果错误消息明细是`reason stack_trace_with_native_method`并且被打印出来的stack trace的顶部是本地方法`native method`,
    这表明本地方法出现了内存分配异常.     
    这和前面的异常不同点在于这个异常是`JNI`(`java native interface`)或者本地方法`native method`中抛出的而不是在JVM中.     
    办法: 看到这个异常需要使用系统工具来诊断这个问题.
