title: 究竟什么是Transaction事务
date: 2015-01-22 12:52
tags:
  - JAVA
categories:
  - JAVA
---

Transaction不管在WEB开发还是数据库中都是相当重要的概念。在WEB开发框架中Transaction的实现方面却有着很多的不同。下面以J2EE介绍Transaction。
# 一、 Transaction是什么

Transaction是指一系列不可分割的改动数据库的操作。这里的“一系列”操作指的是一组SQL语句；“不可分割”就意味着一致性和完整性，要么这一系列操作全部commit（提交），要么就全部rollback（回滚）。（如果一系列的操作只包含enquiry操作，那么这些操作也不是Transaction） 所以常见的代码结构

```java
    public void save(User user) {
    Session s = sessionFactory.openSession();
    s.beginTransaction();
    s.save(user);
    //…
    s.getTransaction().commit();
    s.close();
    }
```
# 二、在J2EE中，Transaction主要有几大类，具体有几种？

在J2EE中，Transaction主要有

* Bean-Managed Transaction
* JDBC Transaction
* JTA Transaction
* Container-Managed Transaction

<!--more-->


# 三、JDBC Transaction

## 定义
JDBC Transaction是指由Database本身去管理的事务。通过显示调用Connection接口的commit和rollback方法来完成事务的提交和回滚。事务结束的边界是commit或者rollback方法的调用，开始的边界于组成当前事务的所有statement中的第一个被执行的时候。
## 使用
使用JDBC Transaction的顺序是获取session（tx = session.beginTransaction()），提交（tx.commit），关闭session（session.close()）
## 代码示例

```java
Session session = sf.openSession(); 
Transaction tx = session.beginTransactioin(); 
... 
session.flush(); 
tx.commit(); 
session.close();
```

# 四、JTA Transaction

## 定义
JTA Transaction是指由J2EE Transaction manager去管理的事务。是调用UserTransaction接口的begin，commit和rollback方法来完成事务范围的界定，事务的提交和回滚。JTA Transaction可以实现同一事务对应不同的数据库，但是它仍然无法实现事务的嵌套。

## 使用
如果需要在EJB中使用Hibernate，或者准备用JTA来管理跨Session的长事务，那么就需要使用JTA Transaction。使用JTA Transaction的顺序是先启动Transaction，然后启动Session，关闭Session，最后提交Transaction

## 代码示例

```java
public void withdrawCash(double amount) {
    UserTransaction ut = context.getUserTransaction();
    try {
       ut.begin();
       updateChecking(amount);
       machineBalance -= amount;
       insertMachine(machineBalance);
       ut.commit();
    } catch (Exception ex) {
        try {
           ut.rollback();
        } catch (SystemException syex) {
            throw new EJBException
               ("Rollback failed: " + syex.getMessage());
        }
        throw new EJBException 
           ("Transaction failed: " + ex.getMessage());
     }
}
```

# 五、Container-Managed Transaction

Container-Managed Transaction，就是由Container（容器）负责管理的Transaction。其最大的特点是不需要显式界定事务的边界，也不需要显式的提交或者回滚事务，这一切都由Container来替我们完成。我们需要做的就是设定在一个Bean中，哪些方法是跟事务相关的，同时设定它们的Transaction Attribute既可。


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

