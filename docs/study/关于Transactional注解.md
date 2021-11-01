#关于Transactional注解

###Transactional注解失效原因查询

1. 方法是不是public；
2. 异常类型是不是unchecked异常

如果想让checked异常也回滚，在注解上面写明异常类型即可:

@Transactional(rollbackFor=Exception.class)

类似还有norollbackFor，自定义不回滚的异常

3. 数据库引擎要支持事务，如果是Mysql，注意表要使用支持事务的引擎，比如innaodb，如果是myisam，事务是不起作用的
4. 是否开启了对注解的解析:

    <tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true"/>

5. spring是否扫描到你使用注解事务的这个类所在的包：

    <context:component-scan base-package="com.xxx.xxx" ></context:component-scan>

6. 检查是不是同一个类中的方法调用(如a方法调用同一个类中的b方法)

7. 异常是不是被catch住了





注：开启事务后不能动态切换数据源