## 1、一次完整的HTTP请求的所经历的步骤

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1681287211024/36710e0040e94402b675356bd5bfe3aa.png)

1、首先进行DNS域名解析（本地浏览器缓存、操作系统缓存或者DNS服务器），首先会搜索浏览器自身的DNS缓存（缓存时间比较短，大概只有1分钟，且只能容纳1000条缓存）

b）如果浏览器自身的缓存里面没有找到，那么浏览器会搜索系统自身的DNS缓存

c）如果还没有找到，那么尝试从 hosts文件里面去找

d）在前面三个过程都没获取到的情况下，就去域名服务器去查找，

2、三次握手建立 TCP 连接

在HTTP工作开始之前，客户端首先要通过网络与服务器建立连接，HTTP连接是通过 TCP 来完成的。HTTP 是比 TCP 更高层次的应用层协议，根据规则，只有低层协议建立之后，才能进行高层协议的连接，因此，首先要建立 TCP 连接，一般 TCP 连接的端口号是80；

3、客户端发起HTTP请求

4、服务器响应HTTP请求

5、客户端解析html代码，并请求html代码中的资源

浏览器拿到html文件后，就开始解析其中的html代码，遇到js/css/image等静态资源时，就向服务器端去请求下载

6、客户端渲染展示内容

7、关闭 TCP 连接

一般情况下，一旦服务器向客户端返回了请求数据，它就要关闭 TCP 连接，然后如果客户端或者服务器在其头信息加入了这行代码 Connection:keep-alive ，TCP 连接在发送后将仍然保持打开状态，于是，客户端可以继续通过相同的连接发送请求，也就是说前面的3到6，可以反复进行。保持连接节省了为每个请求建立新连接所需的时间，还节约了网络带宽。

## 2、讲一讲什么是RPC？RPC在你项目中的运用

RPC（Remote Procedure Call ——远程过程调用），它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络的技术。

一次完整的RPC同步调用流程：

1）服务消费方（client）以本地调用方式调用客户端存根；

2）什么叫客户端存根？就是远程方法在本地的模拟对象，一样的也有方法名，也有方法参数，client stub接收到调用后负责将方法名、方法的参数等包装，并将包装后的信息通过网络发送到服务端；

3）服务端收到消息后，交给代理存根在服务器的部分后进行解码为实际的方法名和参数

4） server stub根据解码结果调用服务器上本地的实际服务；

5）本地服务执行并将结果返回给server stub；

6）server stub将返回结果打包成消息并发送至消费方；

7）client stub接收到消息，并进行解码；

8）服务消费方得到最终结果。

RPC框架的目标就是要中间步骤都封装起来，让我们进行远程方法调用的时候感觉到就像在本地调用一样。

##### 实现RPC框架需要解决的那些问题

###### 代理问题

代理本质上是要解决什么问题？要解决的是被调用的服务本质上是远程的服务，但是调用者不知道也不关心，调用者只要结果，具体的事情由代理的那个对象来负责这件事。既然是远程代理，当然是要用代理模式了。

代理(Proxy)是一种设计模式,即通过代理对象访问目标对象.这样做的好处是:可以在目标对象实现的基础上,增强额外的功能操作,即扩展目标对象的功能。那我们这里额外的功能操作是干什么，通过网络访问远程服务。

jdk的代理有两种实现方式：静态代理和动态代理。

```
public class Client2 {
    //远程调用类
    public static IUserService getStub() throws Exception{
        //创建代理类 
        InvocationHandler handler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket socket = new Socket("127.0.0.1", 8888);
                ByteArrayOutputStream out = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(out);
                dos.writeInt(13);
                socket.getOutputStream().write(out.toByteArray());
                socket.getOutputStream().flush();

                DataInputStream dis = new DataInputStream(socket.getInputStream());
                int ReceId = dis.readInt();
                String name = dis.readUTF();
                User user = new User(ReceId, name);
                dos.close();
                socket.close();
                return user;
            }
        };
        //执行动态代理（传入类加载器、接口、代理对象； 返回对象）
        Object o = Proxy.newProxyInstance(IUserService.class.getClassLoader(),
                new Class[]{IUserService.class},handler);
        return (IUserService)o;
    }
}
```

###### 序列化问题

序列化问题在计算机里具体是什么？我们的方法调用，有方法名，方法参数，这些可能是字符串，可能是我们自己定义的java的类，但是在网络上传输或者保存在硬盘的时候，网络或者硬盘并不认得什么字符串或者javabean，它只认得二进制的01串，怎么办？要进行序列化，网络传输后要进行实际调用，就要把二进制的01串变回我们实际的java的类，这个叫反序列化。java里已经为我们提供了相关的机制Serializable。

###### 登记的服务实例化

登记的服务有可能在我们的系统中就是一个名字，怎么变成实际执行的对象实例，当然是使用反射机制。

反射机制是什么？

反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

反射机制能做什么

反射机制主要提供了以下功能：

•在运行时判断任意一个对象所属的类；

•在运行时构造任意一个类的对象；

•在运行时判断任意一个类所具有的成员变量和方法；

•在运行时调用任意一个对象的方法；

•生成动态代理。

```
public class Client3 {
    //远程调用类
    public static Object getStub(final Class clazz) throws Exception{
        InvocationHandler handler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket socket = new Socket("127.0.0.1", 8888);
                ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
                //TODO 送入的class 灵活了
                String className = clazz.getName();
                String methodName = method.getName();
                Class[] parametersTypes = method.getParameterTypes();
                //TODO 传递class到服务器
                oos.writeUTF(className);

                oos.writeUTF(methodName);
                oos.writeObject(parametersTypes);
                oos.writeObject(args);
                oos.flush();
                //TODO 返回对象
                ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
                Object o = ois.readObject();
                oos.close();
                socket.close();
                return o ;
            }
        };
        Object o = Proxy.newProxyInstance(clazz.getClassLoader(),
                new Class[]{clazz},handler);
        return o;
    }
}
```

```
/**
 * 服务端：服务更灵活-提供多个类、多个方法的远程接口调用
 */
public class Server3 {
    private static boolean running = true;
    public static void main(String[] args) throws  Exception{
        ServerSocket serverSocket = new ServerSocket(8888);
        while (running){
            Socket socket = serverSocket.accept();
            process(socket);
            socket.close();
        }
        serverSocket.close();
    }
    private static void  process(Socket socket) throws Exception{
        InputStream in = socket.getInputStream();
        OutputStream out = socket.getOutputStream();
        ObjectInputStream ois = new ObjectInputStream(in);
        //TODO 拿到客户端传递过来的class
        String clazzName =ois.readUTF();

        String methodName =ois.readUTF();
        Class[] parameterTypes = (Class[])ois.readObject();
        Object[] args =(Object[])ois.readObject();
        //反射拿到class
        Class clazz =Class.forName(clazzName);
        if(clazz.isInterface()){
            if(clazzName.equals("com.msb.netty.pre.IUserService")){
                clazz = UserServiceImpl.class;
            }
           //这里可以使用反射机制拿到所有接口对应的实现类
        }

        Method method = clazz.getMethod(methodName,parameterTypes);

        Object object = method.invoke(clazz.newInstance(),args);
        //TODO 返回值：使用对象进行返回
        ObjectOutputStream oos = new ObjectOutputStream(out);
        oos.writeObject(object);
        oos.flush();
    }
}
```

##### Dubbo是一个典型的RPC运用

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1656916683026/ba2e43a9bfe34909b76e637968dd8304.png)

服务容器负责启动，加载，运行服务提供者。

服务提供者在启动时，向注册中心注册自己提供的服务。

服务消费者在启动时，向注册中心订阅自己所需的服务。

注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。

服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。

服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

## 3、TCP 粘包/拆包的原因及解决方法？

### 什么是粘包和半包？

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1658148713085/ce8627ce64a74002a59162251d94c04f.png)

假设客户端分别发送了两个数据包ABC和DEF给服务端，由于服务端一次读取到的字节数是不确定的，故可能存在以下4种情况

1、服务端分两次读取到了两个独立的数据包，分别是ABC和DDEF，没有粘包和拆包；

2、服务端一次接收到了两个数据包，ABC和DEF粘合在一起，被称为TCP粘包；

3、服务端分两次读取到了两个数据包，第一次读取到了完整的ABC包和DEF包的部分内容(ABCD)，第二次读取到了DEF包的剩余内容(EF)，这被称为TCP拆包

4、服务端分两次读取到了两个数据包，第一次读取到了ABC包的部分内容(AB)，第二次读取到了CD，第三次读到了EF（这种有粘包和半包问题）

### TCP粘包/半包发生的原因

**根本原因：TCP 是流式协议，消息无边界。**
UDP 像邮寄的包裹，虽然一次运输多个，但每个包裹都有“界限”，一个一个签收，所以无粘包、半包问题。

**TCP收发**
一个发送可能被多次接收，多个发送可能被一次接收
**TCP传输**
一个发送可能占用多个传输包，多个发送可能公用一个传输包

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1658148713085/28f961948aad49fc8c159c735cf24f70.png)

### TCP粘包/半包解决

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1658148713085/412f45a5766940c498dc8a3c177e7880.png)

## 5、如何让单机下Netty支持百万长连接？

### 操作系统

首先就是要突破操作系统的限制。
在Linux平台上，无论编写客户端程序还是服务端程序，在进行高并发TCP连接处理时，最高的并发数量都要受到系统对用户单一进程同时可打开文件数量的限制（这是因为系统为每个TCP连接都要创建一个socket句柄，每个socket句柄同时也是一个文件句柄）。
可使用ulimit命令查看系统允许当前用户进程打开的文件数限制：

```
$ ulimit -n 1024
```

这表示当前用户的每个进程最多允许同时打开1024个文件，这1024个文件中还得除去每个进程必然打开的标准输入，标准输出，标准错误，服务器监听 socket,进程间通讯的unix域socket等文件，那么剩下的可用于客户端socket连接的文件数就只有大概1024-10=1014个左右。也就是说缺省情况下，基于Linux的通讯程序最多允许同时1014个TCP并发连接。
对于想支持更高数量的TCP并发连接的通讯处理程序，就必须修改Linux对当前用户的进程同时打开的文件数量。
修改单个进程打开最大文件数限制的最简单的办法就是使用ulimit命令：

```
$ ulimit –n 1000000
```

### Netty调优

设置合理的线程数
对于线程池的调优,主要集中在用于接收海量设备TCP连接、TLS握手的 Acceptor线程池( Netty通常叫 boss NioEventLoop Group)上,以及用于处理网络数据读写、心跳发送的1O工作线程池(Nety通常叫 work Nio EventLoop Group)上。
对于Nety服务端,通常只需要启动一个监听端口用于端侧设备接入即可,但是如果服务端集群实例比较少,甚至是单机(或者双机冷备)部署,在端侧设备在短时间内大量接入时,需要对服务端的监听方式和线程模型做优化,以满足短时间内(例如30s)百万级的端侧设备接入的需要。
服务端可以监听多个端口,利用主从 Reactor线程模型做接入优化,前端通过SLB做4层门7层负载均衡。
主从 Reactor线程模型特点如下:服务端用于接收客户端连接的不再是一个单独的NO线程,而是一个独立的NIO线程池; Acceptor接收到客户端TCP连接请求并处理后(可能包含接入认证等),将新创建的 Socketchanne注册到I/O线程池(subReactor线程池)的某个IO线程,由它负责 Socketchannel的读写和编解码工作; Acceptor线程池仅用于客户端的登录、握手和安全认证等,一旦链路建立成功,就将链路注册到后端 sub reactor线程池的IO线程,由IO线程负责后续的IO操作。
对于IO工作线程池的优化,可以先采用系统默认值(即CPU内核数×2)进行性能测试,在性能测试过程中采集IO线程的CPU占用大小,看是否存在瓶颈， 具体可以观察线程堆栈，如果连续采集几次进行对比,发现线程堆栈都停留在 Selectorlmpl. lockAndDoSelect，则说明IO线程比较空闲,无须对工作线程数做调整。
如果发现IO线程的热点停留在读或者写操作,或者停留在 Channelhandler的执行处,则可以通过适当调大 Nio EventLoop线程的个数来提升网络的读写性能。

### 心跳优化

针对海量设备接入的服务端,心跳优化策略如下。
(1)要能够及时检测失效的连接,并将其剔除,防止无效的连接句柄积压,导致OOM等问题
(2)设置合理的心跳周期,防止心跳定时任务积压,造成频繁的老年代GC(新生代和老年代都有导致STW的GC,不过耗时差异较大),导致应用暂停
(3)使用Nety提供的链路空闲检测机制,不要自己创建定时任务线程池,加重系统的负担,以及增加潜在的并发安全问题。
当设备突然掉电、连接被防火墙挡住、长时间GC或者通信线程发生非预期异常时,会导致链路不可用且不易被及时发现。特别是如果异常发生在凌晨业务低谷期间,当早晨业务高峰期到来时,由于链路不可用会导致瞬间大批量业务失败或者超时,这将对系统的可靠性产生重大的威胁。
从技术层面看,要解决链路的可靠性问题,必须周期性地对链路进行有效性检测。目前最流行和通用的做法就是心跳检测。心跳检测机制分为三个层面：
(1)TCP层的心跳检测,即TCP的 Keep-Alive机制,它的作用域是整个TCP协议栈。
(2)协议层的心跳检测,主要存在于长连接协议中,例如MQTT。
(3)应用层的心跳检测,它主要由各业务产品通过约定方式定时给对方发送心跳消息实现。
心跳检测的目的就是确认当前链路是否可用,对方是否活着并且能够正常接收和发送消息。作为高可靠的NIO框架,Nety也提供了心跳检测机制。
一般的心跳检测策略如下。
(1)连续N次心跳检测都没有收到对方的Pong应答消息或者Ping请求消息,则认为链路已经发生逻辑失效,这被称为心跳超时。
(2)在读取和发送心跳消息的时候如果直接发生了IO异常,说明链路已经失效,这被称为心跳失败。无论发生心跳超时还是心跳失败,都需要关闭链路,由客户端发起重连操作,保证链路能够恢复正常。
Nety提供了三种链路空闲检测机制,利用该机制可以轻松地实现心跳检测
(1)读空闲,链路持续时间T没有读取到任何消息。
(2)写空闲,链路持续时间T没有发送任何消息
(3)读写空闲,链路持续时间T没有接收或者发送任何消息
对于百万级的服务器，一般不建议很长的心跳周期和超时时长。
接收和发送缓冲区调优
在一些场景下,端侧设备会周期性地上报数据和发送心跳,单个链路的消息收发量并不大,针对此类场景,可以通过调小TCP的接收和发送缓冲区来降低单个TCP连接的资源占用率
当然对于不同的应用场景,收发缓冲区的最优值可能不同,用户需要根据实际场景,结合性能测试数据进行针对性的调优

### JVM层面相关性能优化

当客户端的并发连接数达到数十万或者数百万时,系统一个较小的抖动就会导致很严重的后果,例如服务端的GC,导致应用暂停(STW)的GC持续几秒,就会导致海量的客户端设备掉线或者消息积压,一旦系统恢复,会有海量的设备接入或者海量的数据发送很可能瞬间就把服务端冲垮。
JVM层面的调优主要涉及GC参数优化,GC参数设置不当会导致频繁GC,甚至OOM异常,对服务端的稳定运行产生重大影响。
1.确定GC优化目标
GC(垃圾收集)有三个主要指标。
(1)吞吐量:是评价GC能力的重要指标,在不考虑GC引起的停顿时间或内存消耗时,吞吐量是GC能支撑应用程序达到的最高性能指标。
(2)延迟:GC能力的最重要指标之一,是由于GC引起的停顿时间,优化目标是缩短延迟时间或完全消除停顿(STW),避免应用程序在运行过程中发生抖动。
(3)内存占用:GC正常时占用的内存量。
JVM GC调优的三个基本原则如下。
(1) Minor go回收原则:每次新生代GC回收尽可能多的内存,减少应用程序发生Full gc的频率。
2)GC内存最大化原则:垃圾收集器能够使用的内存越大,垃圾收集效率越高,应用程序运行也越流畅。但是过大的内存一次 Full go耗时可能较长,如果能够有效避免FullGC,就需要做精细化调优。
(3)3选2原则:吞吐量、延迟和内存占用不能兼得,无法同时做到吞吐量和暂停时间都最优,需要根据业务场景做选择。对于大多数应用,吞吐量优先,其次是延迟。当然对于时延敏感型的业务,需要调整次序。
2.确定服务端内存占用
在优化GC之前,需要确定应用程序的内存占用大小,以便为应用程序设置合适的内存,提升GC效率。内存占用与活跃数据有关,活跃数据指的是应用程序稳定运行时长时间存活的Java对象。活跃数据的计算方式:通过GC日志采集GC数据,获取应用程序稳定时老年代占用的Java堆大小,以及永久代(元数据区)占用的Java堆大小,两者之和就是活跃数据的内存占用大小。
3.GC优化过程
1、GC数据的采集和研读
2、设置合适的JVM堆大小
3、选择合适的垃圾回收器和回收策略
当然具体如何做，请参考JVM相关课程。而且GC调优会是一个需要多次调整的过程，期间不仅有参数的变化，更重要的是需要调整业务代码。
