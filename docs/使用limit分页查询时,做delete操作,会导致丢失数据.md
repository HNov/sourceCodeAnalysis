# 使用limit分页查询时,做delete操作,会导致丢失数据

[TOC]

## 一、准备数据



### 1.1 mysql数据脚本

```mysql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for test_so_item
-- ----------------------------
DROP TABLE IF EXISTS `test_so_item`;
CREATE TABLE `test_so_item`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `line_no` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `sku` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `qty` decimal(4, 2) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of test_so_item
-- ----------------------------
INSERT INTO `test_so_item` VALUES (1, '00020', 'A0001', 10.00);
INSERT INTO `test_so_item` VALUES (2, '00010', 'NT9531', 1.00);
INSERT INTO `test_so_item` VALUES (3, '00030', 'A0002', 2.00);
INSERT INTO `test_so_item` VALUES (4, '00040', 'A0003', 5.00);


SET FOREIGN_KEY_CHECKS = 1;
```


![](https://gitee.com/HNov/image/raw/master/typora/20200417005231.png)



### 1.2代码

```java
 @Test
    public void test() {
        List<TestSoItem> items = testSoItemService.list();
        //1.当前全量数据
        log.info("1.当前全部数据:{}", items);
        IPage<TestSoItem> page = new Page<>();
        page.setCurrent(1);
        page.setSize(2);
        //2.分页查第一页
        IPage<TestSoItem> items1 = testSoItemService.page(page);
        log.info("2.第一页:{}", JSON.toJSONString(items1));
        //3.删除
        testSoItemService.removeById(items1.getRecords().get(1).getId());
        log.info("3.已删除id:{}", items1.getRecords().get(1).getId());
        //4.add
        TestSoItem addSoItem = new TestSoItem();
        addSoItem.setLineNo("00010");
        addSoItem.setSku("AA0793159");
        addSoItem.setQty(new BigDecimal(1));
        log.info("4:新增记录{}", JSON.toJSONString(addSoItem));
        testSoItemService.save(addSoItem);
        //5.分页查第2页
        page.setCurrent(2);
        IPage<TestSoItem> items2 = testSoItemService.page(page);
        log.info("5.第二页:{}", JSON.toJSONString(items2));
    }
```

## 二、验证

### 1.验证前全部数据

```
<==    Columns: id, line_no, sku, qty
<==        Row: 1, 00020, A0001, 10.00
<==        Row: 2, 00010, NT9531, 1.00
<==        Row: 3, 00030, A0002, 2.00
<==        Row: 4, 00040, A0003, 5.00
<==      Total: 4
```

```json
[TestSoItem(lineNo=00020, sku=A0001, qty=10.00), TestSoItem(lineNo=00010, sku=NT9531, qty=1.00), TestSoItem(lineNo=00030, sku=A0002, qty=2.00), TestSoItem(lineNo=00040, sku=A0003, qty=5.00)]
```

### 2.第一页数据

```
==>  Preparing: SELECT id,line_no,sku,qty FROM test_so_item LIMIT ?,? 
==> Parameters: 0(Long), 2(Long)
<==    Columns: id, line_no, sku, qty
<==        Row: 1, 00020, A0001, 10.00
<==        Row: 2, 00010, NT9531, 1.00
```

```json
{"current":1,"pages":2,"records":[{"id":1,"lineNo":"00020","qty":10.00,"sku":"A0001"},{"id":2,"lineNo":"00010","qty":1.00,"sku":"NT9531"}],"searchCount":true,"size":2,"total":4}
```

### 3.删除记录

```
==>  Preparing: DELETE FROM test_so_item WHERE id=? 
==> Parameters: 2(Long)
<==    Updates: 1
```

### 4. 新增记录

```
==>  Preparing: INSERT INTO test_so_item ( line_no, sku, qty ) VALUES ( ?, ?, ? ) 
==> Parameters: 00010(String), AA0793159(String), 1(BigDecimal)
<==    Updates: 1
```

```json
{"lineNo":"00010","qty":1,"sku":"AA0793159"}
```

### 5.第二页数据

```
==>  Preparing: SELECT id,line_no,sku,qty FROM test_so_item LIMIT ?,? 
==> Parameters: 2(Long), 2(Long)
<==    Columns: id, line_no, sku, qty
<==        Row: 4, 00040, A0003, 5.00
<==        Row: 5, 00010, AA0793159, 1.00
```

```json
{"current":2,"pages":2,"records":[{"id":4,"lineNo":"00040","qty":5.00,"sku":"A0003"},{"id":5,"lineNo":"00010","qty":1.00,"sku":"AA0793159"}],"searchCount":true,"size":2,"total":4}
```

### 6. 验证后的数据

![](https://gitee.com/HNov/image/raw/master/typora/20200417005243.png)





### 7. log

```verilog
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@2e6bac5a] was not registered for synchronization because synchronization is not active
<==    Columns: id, line_no, sku, qty
<==        Row: 1, 00020, A0001, 10.00
<==        Row: 2, 00010, NT9531, 1.00
<==        Row: 3, 00030, A0002, 2.00
<==        Row: 4, 00040, A0003, 5.00
<==      Total: 4
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@232438a8]
JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@7ed49ba] will not be managed by Spring
2020-04-15 17:17:56.950 - INFO 1412 --- [           main] - [] c.z.p.s.impl.TestSoItemServiceImplTest   : 1.当前全部数据:[TestSoItem(lineNo=00020, sku=A0001, qty=10.00), TestSoItem(lineNo=00010, sku=NT9531, qty=1.00), TestSoItem(lineNo=00030, sku=A0002, qty=2.00), TestSoItem(lineNo=00040, sku=A0003, qty=5.00)]


JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@2907d3e8] will not be managed by Spring
 JsqlParserCountOptimize sql=SELECT  id,line_no,sku,qty  FROM test_so_item
<==    Updates: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@2e6bac5a]
==>  Preparing: SELECT COUNT(1) FROM test_so_item 
==> Parameters: 
<==    Columns: COUNT(1)
<==        Row: 4
==>  Preparing: SELECT id,line_no,sku,qty FROM test_so_item LIMIT ?,? 
==> Parameters: 0(Long), 2(Long)
<==    Columns: id, line_no, sku, qty
<==        Row: 1, 00020, A0001, 10.00
<==        Row: 2, 00010, NT9531, 1.00
<==      Total: 2
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@70382eb1]
2020-04-15 17:18:24.496 - INFO 1412 --- [           main] - [] c.z.p.s.impl.TestSoItemServiceImplTest   : 2.第一页:{"current":1,"pages":2,"records":[{"id":1,"lineNo":"00020","qty":10.00,"sku":"A0001"},{"id":2,"lineNo":"00010","qty":1.00,"sku":"NT9531"}],"searchCount":true,"size":2,"total":4}
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@d9420bf] was not registered for synchronization because synchronization is not active
JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@2907d3e8] will not be managed by Spring
==>  Preparing: DELETE FROM test_so_item WHERE id=? 
==> Parameters: 2(Long)
<==    Updates: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@d9420bf]
2020-04-15 17:18:24.597 - INFO 1412 --- [           main] - [] c.z.p.s.impl.TestSoItemServiceImplTest   : 3.已删除id:2
2020-04-15 17:18:24.598 - INFO 1412 --- [           main] - [] c.z.p.s.impl.TestSoItemServiceImplTest   : 4:新增记录{"lineNo":"00010","qty":1,"sku":"AA0793159"}
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@1145d71f] was not registered for synchronization because synchronization is not active
JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@2907d3e8] will not be managed by Spring
==>  Preparing: INSERT INTO test_so_item ( line_no, sku, qty ) VALUES ( ?, ?, ? ) 
==> Parameters: 00010(String), AA0793159(String), 1(BigDecimal)
<==    Updates: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@1145d71f]
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@64aa7a33] was not registered for synchronization because synchronization is not active
JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@2907d3e8] will not be managed by Spring
 JsqlParserCountOptimize sql=SELECT  id,line_no,sku,qty  FROM test_so_item
==>  Preparing: SELECT COUNT(1) FROM test_so_item 
==> Parameters: 
<==    Columns: COUNT(1)
<==        Row: 4
==>  Preparing: SELECT id,line_no,sku,qty FROM test_so_item LIMIT ?,? 
==> Parameters: 2(Long), 2(Long)
<==    Columns: id, line_no, sku, qty
<==        Row: 4, 00040, A0003, 5.00
<==        Row: 5, 00010, AA0793159, 1.00
<==      Total: 2
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@64aa7a33]
2020-04-15 17:18:24.721 - INFO 1412 --- [           main] - [] c.z.p.s.impl.TestSoItemServiceImplTest   : 5.第二页:{"current":2,"pages":2,"records":[{"id":4,"lineNo":"00040","qty":5.00,"sku":"A0003"},{"id":5,"lineNo":"00010","qty":1.00,"sku":"AA0793159"}],"searchCount":true,"size":2,"total":4}
2020-04-15 17:18:24.735 - WARN 1412 --- [      Thread-31] - [] o.s.cloud.stream.binding.BindingService  : Trying to unbind 'sapPurchaseOrder-input', but no binding found.
2020-04-15 17:18:24.736 - INFO 1412 --- [      Thread-31] - [] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2020-04-15 17:18:24.736 - INFO 1412 --- [      Thread-31] - [] o.s.i.channel.PublishSubscribeChannel    : Channel '{server.name}-1.errorChannel' has 0 subscriber(s).
2020-04-15 17:18:24.736 - INFO 1412 --- [      Thread-31] - [] o.s.i.endpoint.EventDrivenConsumer       : stopped _org.springframework.integration.errorLogger
2020-04-15 17:18:24.767 - WARN 1412 --- [      Thread-32] - [] o.s.c.support.DefaultLifecycleProcessor  : Failed to stop bean 'inputBindingLifecycle'

```



## 三、结论

**在使用limit分页查询时,做delete操作,会导致丢失数据。如案例中id=3 的记录,在第一、二页均没有查到。**


![](https://gitee.com/HNov/image/raw/master/typora/20200417005256.png)

![](https://gitee.com/HNov/image/raw/master/typora/20200417005304.png)



## 四、建议

使用Limit分页查询的问题：

1. 数据量比较大时,页数越大，查询性能越差。

   原因参考文章：https://blog.csdn.net/weixin_43066287/article/details/90024600

2. 查询时使用delete 进行物理删除时,会导致漏查询数据（同时更不建议使用物理删除,尽量使用逻辑删除）。