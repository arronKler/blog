

# 文件系统
TODO

### File Descriptors
文件描述符。在POSIX系统上，系统内核会维护一个表示当前已打开的资源和文件的表，每个打开的资源都会有一个分配好的数值标志，也就是文件描述符，通过文件描述符我们就可以对其对应的文件进行读写等操作。windows上虽然和这不一样也使用了类似的概念，node将这两者的不同抽象出来。让我们可以针对任何一个文件都只需要分配和操作它的描述符就可以了。

fs.open会分配一个FD给对应的文件，大部分系统都是限定了文件描述符的数量的，所以打开后要记得关闭掉，否则会一直占用描述符资源，最终可能导致内存泄漏甚至应用垮塌。

[File Descriptor Wikipedia](https://en.wikipedia.org/wiki/File_descriptor)

### 线程池的使用
除了fs.FSWatcher和那些明确的同步操作外，所有的文件系统API都会用到libuv库提供的线程池。具体大小和使用见 [UV_THREADPOOL_SIZE](https://nodejs.org/api/cli.html#cli_uv_threadpool_size_size) 、具体的线程池的文档可见 [libuv thread pool](http://docs.libuv.org/en/latest/threadpool.html)

node线程池的大小是一个固定的，默认是4，如果有的异步API调用需要花费很长时间的话，那么其他API的体验就会降级。一个潜在的解决方法就是设置 UV_THREADPOOL_SIZE 的大小

