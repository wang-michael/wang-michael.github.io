---
layout:     post
title:      JavaRMI相关知识记录
date:       2017-10-07
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础
---
Java RMI 指的是远程方法调用 (Remote Method Invocation)。它是一种机制，能够让在某个 Java 虚拟机上的对象调用另一个 Java 虚拟机中的对象上的方法。通过Java RMI系统的封装，我们无需关注对象调用过程中的网络数据传输细节，大大简化了我们的开发过程。
### **1. Java RMI简单示例**  
&emsp;&emsp;Server端:
```java
//接口继承自Remote，接口的所有方法都必须声明可抛出RemoteException
public interface RemoteHelloWord extends Remote {
    String sayHello() throws RemoteException;
}

public class RemoteHelloWordImpl implements RemoteHelloWord {    

    //接口实现类，含有将要被客户端远程调用的sayHello方法
    @Override
    public String sayHello() throws RemoteException {
        System.out.println("server: " + name);
        return "Hello World!";
    }  
}

public class RMIServer {

    public static void main(String[] args) {
        //设置此参数保存JDK生成的代理类
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        //设置此参数可在获取Registry时使用JDK动态代理，若不设置，则会在客户端使用RegistryImpl_Stub类代理，服务端使用RegistryImpl_Skel类代理
        System.setProperty("java.rmi.server.ignoreStubClasses", "true");
        try {
            RemoteHelloWord hello1 = new RemoteHelloWordImpl("hello1");
            //调用exportObject方法导出hello1对象,参数0意味着实际导出端口将在运行时由RMI系统或底层操作系统自行指定
            //stub1实际是JDK生成的动态代理类对象
            RemoteHelloWord stub1 = (RemoteHelloWord) UnicastRemoteObject.exportObject(hello1, 0);
            RemoteHelloWord hello2 = new RemoteHelloWordImpl("hello2");
            //调用exportObject方法导出hello2对象,导出对象使用的端口与hello1相同
            //stub2实际是JDK生成的动态代理类对象
            RemoteHelloWord stub2 = (RemoteHelloWord) UnicastRemoteObject.exportObject(hello2, 0);
                      
            LocateRegistry.createRegistry(1099);
            //由于上面设置了java.rmi.server.ignoreStubClasses参数，此处获取的registry实际是JDK动态代理生成的registry对象
            Registry registry=LocateRegistry.getRegistry("localhost");
            //调用的实际是RegistryImpl类中的方法，类中维护了一个HashTable，保存注册的名称与对象之间的映射关系
            registry.bind("helloword1", stub1);
            registry.bind("helloword2", stub2);            
        } catch (RemoteException e) {
            e.printStackTrace();
        } catch (AlreadyBoundException e) {
            e.printStackTrace();
        }
    }
}
	
```   
**问题：**客户端多次获取到的服务器端导出的对象对应的都是服务器端的同一个导出的对象吗？如果这样的话会不会存在线程安全问题呢？？？    

&emsp;&emsp;Client端:  
```java
//注意接口所在包名要与server端相同
public interface RemoteHelloWord extends Remote {
    String sayHello() throws RemoteException;   
}

public class RMIClient {

    public static void main(String[] args) {
        //设置此参数保存JDK生成的代理类
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        //设置此参数可在获取Registry时使用JDK动态代理，若不设置，则会在客户端使用RegistryImpl_Stub类代理，服务端使用RegistryImpl_Skel类代理
		System.setProperty("java.rmi.server.ignoreStubClasses", "true");
        try {
            //使用JDK动态代理获取Registry对象
            Registry registry = LocateRegistry.getRegistry("localhost");          
            //获取的实际是JDK动态代理生成的RemoteHelloWord类的代理对象
            RemoteHelloWord hello2 = (RemoteHelloWord) registry.lookup("helloword2");
            //代理对象与服务器远程对象沟通，沟通的端口为服务器导出远程对象使用的端口，调用的实际是远程对象的sayHello方法，代理对象屏蔽了细节
            hello2.sayHello();
        } catch (RemoteException e) {
            e.printStackTrace();
        } catch (NotBoundException e) {
            e.printStackTrace();
        }
    }
}
```
