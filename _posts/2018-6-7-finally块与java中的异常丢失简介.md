---
layout:     post
title:      finally块与java中的异常丢失简介
date:       2018-6-7
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础
---
### **java中的异常丢失**  
直接上例子：
```java
public class TestException {
    public TestException() {
    }

    boolean testEx() throws Exception {
        boolean ret = true;
        try {
            ret = testEx1();
        } catch (Exception e) {
            System.out.println("testEx, catch exception");
            ret = false;
            throw e;
        } finally {
            System.out.println("testEx, finally; return value=" + ret);
            return ret;
        }
    }

    boolean testEx1() throws Exception {
        boolean ret = true;
        try {
            ret = testEx2();
            if (!ret) {
                System.out.println("testEx1 return false");
                return false;
            }
            System.out.println("testEx1, at the end of try");
            return ret;
        } catch (Exception e) {
            System.out.println("testEx1, catch exception");
            ret = false;
            throw e;
        } finally {
            System.out.println("testEx1, finally; return value=" + ret);
            return !ret;
        }
    }

    boolean testEx2() throws Exception {
        boolean ret = true;
        try {
            int b = 12;
            int c;
            for (int i = 2; i >= -2; i--) {
                c = b / i;
                System.out.println("i=" + i);
            }
            return true;
        } catch (Exception e) {
            System.out.println("testEx2, catch exception");
            ret = false;
            throw e;
        } finally {
            System.out.println("testEx2, finally; return value=" + ret);
            // 1：上面catch块中抛出的异常会丢失
            return ret;
        }
    }

    public static void main(String[] args) {
        TestException testException1 = new TestException();
        try {
            testException1.testEx();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
这里异常之所以会丢失的原因是finally块中的返回值覆盖了抛出的异常。

---
### **finally块与try-catch块中返回语句执行顺序**
try-catch中返回有两种情况：  
* 抛出异常，本层try-catch块不能捕获
* 执行到了return语句

对于第一种情况，finally块中语句是在异常抛出之后，交给外层语句处理之前被执行，并且finally块中有return语句会覆盖之前抛出的异常。如下例：  
```java
ReturnObject testEx() throws Exception {
    ReturnObject returnObject = new ReturnObject();
    returnObject.k = 2;
    int temp = 0;
    try {
        try {
            System.out.println("testEx, at the end of try");
            throw new Exception();
        } catch (Exception e) {
            System.out.println("testEx, catch exception");
            temp = 1;
            // 异常在向本层try-catch语句之外抛出前会执行finally块
            throw e;
        } finally {
            System.out.println("testEx, finally;");
            returnObject.k = 3;
            temp = 2;
        }
    } catch (Exception e) {
        System.out.println("外层 catch " + returnObject.k + " temp: " + temp);
        return returnObject;
    }
}
class ReturnObject {
    int k = 1;
}
```
对于第二种情况，finally块是在return语句执行之后，返回之前执行，如果finally中有return语句，则会覆盖之前的返回值。  

如果finally块中没有return语句且try-catch中的return返回值是基本类型或基本类型的包装类型，finally块中语句对于try-catch中return的返回值的操作是无效的； 如下例：  
```java
boolean testEx() throws Exception {
    boolean ret = true;
    try {
        ret = testEx1();
    } catch (Exception e) {
        System.out.println("testEx, catch exception");
        ret = false;
        throw e;
    } finally {
        System.out.println("testEx, finally; return value=" + ret);
        return ret;
    }
}

Boolean testEx1() throws Exception {
    Boolean ret = new Boolean(true);
    try {
        System.out.println("testEx1, at the end of try");
        return ret;
    } catch (Exception e) {
        System.out.println("testEx1, catch exception");
        ret = false;
        throw e;
    } finally {
        System.out.println("testEx1, finally; return value=" + ret);
        // finally块中对于ret值的更改在testEx方法中不可见
        ret = false;
    }
}
``` 
如果finally块中没有return语句且try-catch中的return返回值不属于基本类型或基本类型的包装类型，finally块中语句对于try-catch中return的返回值对象的操作是可见的；如下例：
```java
boolean testEx() throws Exception {
    boolean ret = true;
    try {
        ReturnObject returnObject = testEx1();
        System.out.println("testEx: " + returnObject.k);
    } catch (Exception e) {
        System.out.println("testEx, catch exception");
        ret = false;
        throw e;
    } finally {
        System.out.println("testEx, finally; return value=" + ret);
        return ret;
    }
}

ReturnObject testEx1() throws Exception {
    ReturnObject returnObject = new ReturnObject();
    returnObject.k = 2;
    try {
        System.out.println("testEx, at the end of try");
        return returnObject;
    } catch (Exception e) {
        System.out.println("testEx, catch exception");
        // 异常在向本层try-catch语句之外抛出前会执行finally块
        throw e;
    } finally {
        System.out.println("testEx, finally;");
        // 对于returnObject的更改是可见的
        returnObject.k = 3;
        // 对象引用的值已经放到方法返回值的栈上了，这是将returnObject指向新的引用并不会改变方法返回栈上的值
        returnObject = new ReturnObject();
    }
}

class ReturnObject {
    int k = 1;
}

public static void main(String[] args) {
    TestException testException1 = new TestException();
    try {
        testException1.testEx();
    } catch (Exception e) {
        e.printStackTrace();
    }
}

输出： 
testEx, at the end of try
testEx, finally;
testEx: 3
testEx, finally; return value=true
```
其实上面两种情况的原因都是因为return语句执行之后返回值已经放到方法返回值的栈顶了，如果返回值是一个对象引用，那么finally块中对于这个引用内部进行的一些操作还是可见的，如果尝试将返回的引用替换为另一个引用，外部是不可见的，因为新的引用并没有放到方法返回值的栈顶，除非使用return语句覆盖try-catch中的return语句，将变化后的引用返回。    

---
### **finally块不被执行的几种情况**  
* 与finally块对应的try块并没有被执行到
* 在try-catch块中含有System.exit(0)语句
* 正在执行try-catch块中代码的线程是守护线程，并且执行过程中守护线程退出

(完)