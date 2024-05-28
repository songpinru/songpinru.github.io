# Buffer

* IntBuffer
* FloatBuffer
* CharBuffer
* DoubleBuffer
* ShortBuffer
* LongBuffer
* ByteBuffer

指针

* position：当前指针
* limit：可用的末尾指针（超过部分不可访问）
* capacity：容量，不可变
* mark：标记，可以使用reset回到该位置

方法：

* put：增加数据，position后移，增加后不可超过limit
* flip：翻转指针，limit=position，position=0
* get：读取数据，position后移，读取不可超过limit
* rewind：position重置为0
* mark：标记，配合reset回到该位置
* clear：position置0，limit=capacity



ByteBuffer:

* slice：切片
* 

# Channel

* FileChannel
* SocketChannel
* ServerSocketChannel
* DatagramChannel

## 获取：

1. 使用getChannel()方法

* FileInputStream/FileOutputStream
* RandomAccessFile
* Socket
* ServerSocket
* DatagramSocket 

2. open()方法

3. Files.newByteChannel()

## FileChannel

零拷贝

```java
FileChannel inChannel = FileChannel.open(Paths.get("/path/to/src"));
FileChannel outChannel = FileChannel.open(Paths.get("/path/to/destination"));
inChannel.transferTo(0,inChannel.size(),outChannel);
```

常用类：

* Files，针对文件的所有操作
* Path，Paths
* StandardOpenOption:  操作描述符
* FileSystems

方法：

* read：
  * 分散读取（有多个buffer）
  *  ==read之后一定要flip==
  *  read前不clear，数据会重复
* write：
  * ==write前一定要flip==
  * 聚集写入（有多个buffer）
* size：这个channel的size，即文件大小
* position：当前Channel的偏移（offset）
* force：强制刷新到磁盘
* transferTo:把数据导入另一个Channel，可以实现零拷贝，mmap+write
  * direct（零拷贝） -> mapped -> jvm内存
  * direct零拷贝最大写入2G
  * transferTo一次最大写入2G，如果使用内存映射，此处内存映射限制是8M，while循环，直到2G拷完）
  * 如果是socket，最大写入8M（SelChImpl的子类，执行一次会自动跳出），也就是说除了file都是2G
  * 返回值为本次写入的字节数（scoket最大8M，file最大2G）
* transferFrom：从另一个channel导入
  * 不能实现零拷贝
  * 如果是file2file，使用内存映射（8M）
  * 如果不是file2file，使用堆内内存
  * 没有读取限制，可以一次读完
  * 如果是scoket，这里的position是当前的，读取过的不算
* map：内存文件映射，网上有用来做零拷贝，但是最大限制2G，可以用来做修改文件
* truncate：截断，保留相应大小的前面部分，后面丢弃
* lock：给文件加锁，阻塞
* tryLock：给文件加锁，非阻塞，不成功返回null

### Charset:

```java
Charset charset = Charset.forName("UTF-8");
CharBuffer allocate = CharBuffer.allocate(1024);
ByteBuffer encode = charset.encode(allocate);
CharBuffer decode = charset.decode(encode);
```

## Selector

Selector是操作系统帮我们管理socket的文件描述符的管理器，把channel注册上去就可以交给系统管理了

selector只接受非阻塞的channel

select方法阻塞实际上是底层SelectionKey阻塞，如果select方法执行时没有的key，是不会触发唤醒的。

使用wakeup唤醒

* keys：注册的key的集合
* selectedKeys：有事件的key的集合，也就是keys的子集
  * 这里的keys显示的是已经触发的，如果没有手动删除的话，会包含之前触发的key（就算删了新触发的也会写进去），但是不会重复触发
* interestOps：注册的事件选项
* readyOps：已触发的事件选项

## SocketChannel

```java
SocketChannel client = SocketChannel.open();
client.bind(null);//绑定本地端口，null为自动绑定，
client.connect(InetSocketAddress.createUnresolved("localhost",8080));
client.configureBlocking(false);//非阻塞
Selector selector = Selector.open();
SelectionKey selectionKey = client.register(selector, SelectionKey.OP_CONNECT);
//selectionKey.attach()//附加对象
//selectionKey.attachment()//获取附加对象
selectionKey.interestOps(SelectionKey.OP_ACCEPT);//等于注册
selectionKey.interestOpsAnd(SelectionKey.OP_READ);//条件做AND
selectionKey.interestOpsOr(SelectionKey.OP_WRITE);//OR

selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
selectionKey.isValid();

selectionKey.readyOps();//就绪的ops集合，即注册的ops
```

## ServerSocketChannel

```java
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(InetSocketAddress.createUnresolved("localhost",8080));
server.configureBlocking(false);//非阻塞
// Selector selector = Selector.open();
server.register(selector,SelectionKey.OP_ACCEPT);


SocketChannel socketChannel = server.accept();//获取通道
socketChannel.configureBlocking(false);//非阻塞

```

常用类

* StandardProtocolFamily：网络协议（ipv4 | ipv6）

## DatagramChannel

```java
DatagramChannel datagramChannel = DatagramChannel
        .open(StandardProtocolFamily.INET)
        .bind(InetSocketAddress.createUnresolved("localhost", 8080));
datagramChannel.configureBlocking(false);
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
datagramChannel.receive(byteBuffer);
datagramChannel.send(byteBuffer,InetSocketAddress.createUnresolved("localhost",8080));
datagramChannel.register(selector,SelectionKey.OP_ACCEPT);
```

## Pipe

Pipe是线程间的管道，用来异步通信

```java
Pipe pipe = Pipe.open();
Pipe.SourceChannel source = pipe.source();
Pipe.SinkChannel sink = pipe.sink();
source.keyFor(selector);
source.register(selector,SelectionKey.OP_ACCEPT);
source.configureBlocking(false);
sink.keyFor(selector);
ByteBuffer buffer = ByteBuffer.allocate(1024);
source.read(buffer);
sink.write(buffer);
```