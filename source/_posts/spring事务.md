---
title: spring事务
date: 2026-09-01 21:42:22
tags: 技术
---

# Java 中的 MySQL 事务：声明式事务与编程式事务

## 一、事务的本质

事务用于把多条数据库操作组织成一个不可分割的执行单元：

```
全部成功 → COMMIT
任意失败 → ROLLBACK
```

以转账为例：

```
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
```

两条 SQL 必须使用**同一个数据库连接**并处于同一个事务中，否则无法统一提交或回滚。

Java 本身不负责实现数据库事务。它通过 JDBC 驱动向 MySQL 发送事务控制请求：

```
Java/Spring决定事务边界
        ↓
JDBC设置autoCommit并发送COMMIT或ROLLBACK
        ↓
MySQL/InnoDB执行锁、MVCC、undo、redo和日志提交
```

关闭自动提交后，不必显式执行 `BEGIN`：

```
connection.setAutoCommit(false);

updateA();           // 第一条事务性SQL隐式开启事务
updateB();           // 加入当前事务
connection.commit(); // 结束当前事务

updateC();           // 隐式开启下一个事务
connection.rollback();
```

如果保持默认的 `autocommit=true`，每条 SQL 都是一个独立事务：

```
UPDATE A → 开启 → 执行 → 提交
UPDATE B → 开启 → 执行 → 提交
```

因此，`autocommit=true` 并不是没有事务，而是一条语句一个事务。

------

## 二、声明式事务

声明式事务通过 `@Transactional` 声明事务边界：

```
@Service
public class TransferService {

    private final AccountMapper accountMapper;

    public TransferService(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
    }

    @Transactional(rollbackFor = Exception.class)
    public void transfer(Long from, Long to, BigDecimal amount) {
        int deducted = accountMapper.deduct(from, amount);
        if (deducted == 0) {
            throw new IllegalStateException("余额不足");
        }

        accountMapper.add(to, amount);
    }
}
```

Spring 使用 AOP 代理包装方法，整体流程近似于：

```
beginTransaction();

try {
    Object result = target.transfer(...);
    commitTransaction();
    return result;
} catch (Throwable e) {
    rollbackTransaction();
    throw e;
}
```

事务在进入代理方法时开启，在**整个方法正常执行结束后**提交，而不是最后一条 SQL 执行完后立即提交。

```
@Transactional
public void createOrder() {
    orderMapper.insert(...);
    stockMapper.decrease(...);

    callRemoteService(); // 此时事务仍未提交
} // 方法正常结束后提交
```

这意味着远程调用期间，数据库连接和行锁可能一直被占用。因此事务方法中应尽量避免网络请求、大量计算和长时间等待。

### 声明式事务的回滚规则

Spring 默认对 `RuntimeException` 和 `Error` 回滚。希望受检异常也触发回滚，可以指定：

```
@Transactional(rollbackFor = Exception.class)
```

如果异常被内部捕获并且没有继续抛出，方法仍然正常返回，事务可能提交：

```
@Transactional
public void createOrder() {
    try {
        orderMapper.insert(...);
        throw new IOException();
    } catch (Exception e) {
        log.error("执行失败", e); // 异常被吞掉
    }
}
```

需要重新抛出异常，或者显式标记当前事务只能回滚：

```
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

### 事务失效场景

#### 1. 代理有没有生效

`@Transactional` 依赖 AOP 代理，因此：

- 同类内部调用；
- 自己 `new` 对象；
- `private/static/final` 等无法正常代理的方法；

都会导致调用没有经过事务拦截器。

```
public void outer() {
    inner(); // 同类内部调用
}

@Transactional
public void inner() {
}
```

#### 2. 异常有没有触发回滚

事务可能已经开启，但没有按预期回滚：

- 异常被 `catch` 吞掉；
- 默认受检异常不回滚；
- 配置了错误的 `rollbackFor/noRollbackFor`。

```
@Transactional
public void save() {
    try {
        mapper.insert(...);
    } catch (Exception e) {
        // 吞掉异常，方法正常返回
    }
}
```

#### 3. 是否还在同一个线程

Spring 通常通过 `ThreadLocal` 保存事务连接，因此：

- `@Async`；
- `CompletableFuture`；
- 手动创建线程；

不会自动继承原事务。

```
@Transactional
public void save() {
    CompletableFuture.runAsync(() -> mapper.insert(...));
}
```

#### 4. 是否使用同一个连接

一个本地事务只能控制一个事务连接：

- 手动创建 JDBC Connection；
- 使用另一个数据源；
- 使用未被 Spring 管理的 `SqlSession`；

都可能脱离当前事务。

#### 5.数据库是否支持回滚

即使 Spring 层完全正确，数据库层也可能不满足条件：

- MyISAM 不支持事务；
- DDL 可能隐式提交；
- 跨数据库需要分布式事务；
- 错误的事务传播行为可能挂起或新建事务。

------

## 三、编程式事务

编程式事务由代码主动定义事务范围。Spring 推荐使用 `TransactionTemplate`：

```
@Service
public class OrderService {

    private final OrderMapper orderMapper;
    private final StockMapper stockMapper;
    private final TransactionTemplate transactionTemplate;

    public OrderService(
            OrderMapper orderMapper,
            StockMapper stockMapper,
            TransactionTemplate transactionTemplate) {
        this.orderMapper = orderMapper;
        this.stockMapper = stockMapper;
        this.transactionTemplate = transactionTemplate;
    }

    public void createOrder(Order order) {
        validateOrder(order); // 不在事务中

        transactionTemplate.executeWithoutResult(status -> {
            orderMapper.insert(order);

            int affected =
                    stockMapper.decreaseStock(order.getProductId());

            if (affected == 0) {
                throw new IllegalStateException("库存不足");
            }
        });

        sendNotification(order); // 不在事务中
    }
}
```

只有 Lambda 中的代码属于事务：

```
普通校验
    ↓
开启事务
    ↓
插入订单
扣减库存
    ↓
提交事务
    ↓
发送通知
```

这种方式适合精确缩小事务范围，避免在事务中执行耗时操作。

### 返回事务结果

```
Long orderId = transactionTemplate.execute(status -> {
    orderMapper.insert(order);
    return order.getId();
});
```

回调正常结束后，Spring 先提交事务，再从 `execute()` 返回结果。

### 主动回滚

第一种方式是抛出运行时异常：

```
transactionTemplate.executeWithoutResult(status -> {
    orderMapper.insert(order);

    if (stockMapper.decreaseStock(productId) == 0) {
        throw new IllegalStateException("库存不足");
    }
});
```

第二种方式是标记只能回滚：

```
Boolean success = transactionTemplate.execute(status -> {
    orderMapper.insert(order);

    if (stockMapper.decreaseStock(productId) == 0) {
        status.setRollbackOnly();
        return false;
    }

    return true;
});
```

`setRollbackOnly()` 后应立即结束当前逻辑。它不会终止代码执行，只是确保事务最终不能提交。

### 直接使用事务管理器

也可以直接使用 `PlatformTransactionManager`：

```
TransactionStatus status =
        transactionManager.getTransaction(new DefaultTransactionDefinition());

try {
    orderMapper.insert(order);
    stockMapper.decreaseStock(productId);
    transactionManager.commit(status);
} catch (Exception e) {
    transactionManager.rollback(status);
    throw e;
}
```

这种方式控制更细，但更容易遗漏提交、回滚和资源清理，普通业务通常优先选择 `TransactionTemplate`。

------

## 四、MyBatis-Plus 如何加入事务

MyBatis-Plus 没有独立实现事务。Spring 开启事务后会：

```
1. 从DataSource取得Connection
2. 设置autoCommit=false
3. 将Connection绑定到当前线程
4. 执行业务代码
```

MyBatis-Plus 的多个 Mapper 会取得当前线程绑定的同一个连接：

```
OrderMapper ─┐
             ├── 同一个Connection、同一个MySQL事务
StockMapper ─┘
```

因此，只要满足以下条件，多个 Mapper 操作就能统一提交或回滚：

- 使用同一个数据源；
- 在同一个线程中执行；
- 位于同一个事务边界内；
- 没有手动创建其他 JDBC 连接。

MyBatis-Plus 的批量操作也可以放入事务模板：

```
transactionTemplate.executeWithoutResult(status -> {
    userService.saveBatch(users);

    if (checkFailed()) {
        throw new IllegalStateException("批量保存失败");
    }
});
```

------

## 五、事务为什么不能自动解决读后写竞争

事务保证原子提交，但不代表事务之间完全串行执行。

```
Product product = productMapper.selectById(id);

if (product.getStock() > 0) {
    product.setStock(product.getStock() - 1);
    productMapper.updateById(product);
}
```

两个事务可能同时读到：

```
stock = 1
```

随后都根据旧值写入 `stock=0`，最终创建了两个订单。两个事务内部都正常执行，因此事务机制不会主动回滚任何一个。

应使用原子条件更新：

```
UPDATE product
SET stock = stock - 1
WHERE id = ?
  AND stock > 0;
```

然后检查影响行数：

```
int affected = productMapper.decreaseStock(productId);
if (affected == 0) {
    throw new IllegalStateException("库存不足");
}
```

复杂业务也可以使用：

```
SELECT * FROM product WHERE id = ? FOR UPDATE;
```

或者通过版本号实现乐观锁。

事务解决的是“多个操作能否作为整体提交”，锁和并发控制解决的是“多个事务同时操作相同数据时如何协调”。

------

## 六、两种事务方式如何选择

| 场景                                 | 推荐方式                     |
| ------------------------------------ | ---------------------------- |
| 整个 Service 方法都需要事务          | `@Transactional`             |
| 只让方法中的一小段代码进入事务       | `TransactionTemplate`        |
| 需要根据业务条件主动回滚             | 两者均可，编程式更直观       |
| 需要动态设置传播行为、超时或隔离级别 | `TransactionTemplate`        |
| 项目中事务方法非常多                 | 声明式事务                   |
| 需要完全控制提交和回滚时机           | `PlatformTransactionManager` |

声明式事务更简洁，适合稳定、清晰的方法级事务；编程式事务更灵活，适合事务范围需要精确控制的业务。二者底层最终都依赖同一个 Spring 事务管理器、JDBC Connection 和 MySQL/InnoDB 事务机制。
