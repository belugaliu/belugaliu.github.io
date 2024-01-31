---
title: Redis 分布式锁你续约了吗
date: 2024-01-31 21:38:27
categories:
- 分布式锁
tags:
- 分布式锁
- redis
---
服务在集群情况下，线程锁是无法满足服务之间逻辑隔离。分布式锁概念应运而生，它需要具备互斥性、防止死锁、高可用性、可重入性、唯一标识的特点。

- 互斥性：任意时刻，只能有一个服务才能获取锁。

- 防止死锁：分布式锁应该在服务逻辑运行异常或崩溃时能够自动释放。一般的做法是给锁设定超时时间避免死锁。

- 高可用性：确保锁提供方节点故障时也能正常工作，确保锁的可靠性。

- 可重入性：允许同一个线程或服务在持有锁的情况下多次获取同一个锁，而不会出现死锁或阻塞。

- 唯一标识：分布式锁应该具备唯一的标识。

分布式锁方案大体有几种，使用基于唯一索引的数据库表、zookeeper/etcd、redis。为达到分布式锁的互斥性和防止死锁这两个特性，方案是设定超时时间配合**定时续期**以达到目的。如果你用 Redis 实现分布式锁，请问你项目中 Redis 分布式锁有**定时续期**吗？

Jedis 主要包含数据结构操作和队列 PUB/SUB 操作。Redisson 组件除此之外还包含分布式锁的实现。Redisson 关于获取锁有六种方式，`lock()` 、`tryLock()`、`lockInterruptibly()` 、`tryLock(long waitTime, TimeUnit unit)`、 `lock(long leaseTime, TimeUnit unit)` 、`tryLock(long waitTime, long leaseTime, TimeUnit unit)`。区分点在于 是否等待获取锁、等待获取锁时长，是否有过期、过期时间。好像没有看到续期相关的内容。就像老夫老妻一样，:heart: 爱是藏在细节里，续期藏在锁获取的细节里。

> :vertical_traffic_light: 代码是以 redisson-3.18.0 版本为例，估计总体思路差不多。

```java
// RedissonLock
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    RFuture<Long> ttlRemainingFuture;
    if (leaseTime > 0) {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    }
    CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
        // lock acquired
        if (ttlRemaining == null) {
            if (leaseTime > 0) {
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                scheduleExpirationRenewal(threadId);
            }
        }
        return ttlRemaining;
    });
    return new CompletableFutureWrapper<>(f);
}   
```

首先，没有设置过期时间时，redisson 会使用 `internalLockLeaseTime` （它指 lock 内置过期时间，lock 对象初始化时从配置类 `org.redisson.config.Config#lockWatchdogTimeout` 中获取，默认 30 s）作为过期时间来申请分布式锁。第一次申请锁成功后 `ttlRemainingFuture.thenApply` ，如果自定义过期时间有值，则重新设置 `internalLockLeaseTime`。没有设置的话，则需要定时续期，保证锁能被本线程一直持有。

```java
// RedissonBaseLock
    protected void scheduleExpirationRenewal(long threadId) {
    ExpirationEntry entry = new ExpirationEntry();
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    if (oldEntry != null) {
        // 可重入位置，因为不存在获取锁之后，同一线程的并发问题，所以这里使用 LinkedHashMap
        oldEntry.addThreadId(threadId);
    } else {
        entry.addThreadId(threadId);
        try {
            renewExpiration();
        } finally {
            if (Thread.currentThread().isInterrupted()) {
                cancelExpirationRenewal(threadId);
            }
        }
    }
}
private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }
    Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
            if (ent == null) {
                return;
            }
            Long threadId = ent.getFirstThreadId();
            if (threadId == null) {
                return;
            }
            CompletionStage<Boolean> future = renewExpirationAsync(threadId);
            future.whenComplete((res, e) -> {
                if (e != null) {
                    log.error("Can't update lock " + getRawName() + " expiration", e);
                    EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                    return;
                }
                if (res) {
                    // reschedule itself
                    renewExpiration();
                } else {
                    cancelExpirationRenewal(null);
                }
            });
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    ee.setTimeout(task);
}
```

**简单概括，就是未设置过期时间的分布式锁，是以 30s 过期时间先获取分布式锁，程序中使用时间片方式在每 10s (30s * 1/3) 续期。设置过期时间的分布式锁反而不能享受续期。**

:sweat_smile: 本来大家在实战过程中，就是怕锁不释放，基本上都会被建议使用 `tryLock(long waitTime, long leaseTime, TimeUnit unit)` ，结果只有这种设置 leaseTime 的获取锁没有续期。多少有点被背 (feng) 刺 (ci) :bee:。我们需要解决这个问（feng）题（ci），希望在 leaseTime 设置时，也能享受续期功效。

```java
package org.redisson;

import java.lang.reflect.Field;
import java.util.Objects;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RFuture;
import org.redisson.api.RLock;

/**
 * 分布式锁.<br>
 * 用装饰者模式给所有 lock 加上时间过期前的续期操作.<br>
 * redis 作为分布式锁方案一定会有弊端，比如出现哨兵模式的 redis 集群，就可能因为锁信息在主节点同步从节点时出现的主节点中断，导致从节点成为主节点之后
 * 无锁信息，导致的其他线程申请到锁，此时就会出现两个线程获取到同一把锁的 ganga 场景.<br>
 * 请注意合理使用锁，获取锁后一定在 finally 中释放锁，程序运行时间长之后一定会出现内存溢出问题.
 *
 * @author liulili
 * @since 20243-01-19
 */
@Slf4j
public class RenewalLock implements RLock {

    private final RedissonBaseLock lock;

    public RenewalLock(RLock lock) {
        if (lock instanceof RenewalLock) {
            throw new IllegalArgumentException("lock 本就是续期锁，不需要二次装饰！");
        }
        if (lock instanceof RedissonBaseLock) {
            this.lock = (RedissonBaseLock) lock;
            return;
        }
        throw new IllegalArgumentException("分布式锁续期的功能至少是 RedissonBaseLock 的实例，到 redisson 3.18.0 版本，RedissonMultiLock 是不能被 DistributedLock 装饰的！");
    }

    @Override
    public String getName() {
        return "renewal_" + lock.getName();
    }

    private void scheduleExpirationRenewal(long leaseTime) {
        this.scheduleExpirationRenewal(leaseTime, Thread.currentThread().getId());
    }

    private void scheduleExpirationRenewal(long leaseTime, long threadId) {
        try {
            Field field = RedissonBaseLock.class.getDeclaredField("internalLockLeaseTime");
            field.setAccessible(true);
            field.setLong(lock, leaseTime);
        } catch (Exception e) {
            log.error("RedissonBaseLock 类没有 internalLockLeaseTime 属性，请注意版本", e);
        }
        lock.scheduleExpirationRenewal(threadId);
    }
    private void cancelExpirationRenewal() {
        this.cancelExpirationRenewal(Thread.currentThread().getId());
    }
    private void cancelExpirationRenewal(long threadId) {
        lock.cancelExpirationRenewal(threadId);
    }

    @Override
    public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
        lock.lockInterruptibly(leaseTime, unit);
        if (leaseTime > -1) {
            this.scheduleExpirationRenewal(leaseTime);
        }
    }

    @Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        boolean getLockSuccess = lock.tryLock(waitTime, leaseTime, unit);
        if (getLockSuccess && leaseTime > -1) {
            this.scheduleExpirationRenewal(leaseTime);
        }
        return getLockSuccess;
    }

    @Override
    public void lock(long leaseTime, TimeUnit unit) {
        lock.lock(leaseTime, unit);
        if (leaseTime > -1) {
            this.scheduleExpirationRenewal(leaseTime);
        }
    }

    @Override
    public boolean forceUnlock() {
        try {
            return lock.forceUnlock();
        } finally {
            this.cancelExpirationRenewal();
        }

    }

    @Override
    public boolean isLocked() {
        return lock.isLocked();
    }

    @Override
    public boolean isHeldByThread(long threadId) {
        return lock.isHeldByThread(threadId);
    }

    @Override
    public boolean isHeldByCurrentThread() {
        return lock.isHeldByCurrentThread();
    }

    @Override
    public int getHoldCount() {
        return lock.getHoldCount();
    }

    @Override
    public long remainTimeToLive() {
        return lock.remainTimeToLive();
    }

    // 还是默认的超时时间
    @Override
    public void lock() {
        lock.lock();
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        lock.lockInterruptibly();
    }

    @Override
    public boolean tryLock() {
        return lock.tryLock();
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return tryLock(time, -1, unit);
    }

    @Override
    public void unlock() {
        try {
            lock.unlock();
        } finally {
            this.cancelExpirationRenewal();
        }

    }

    @Override
    public Condition newCondition() {
        return lock.newCondition();
    }

    @Override
    public RFuture<Boolean> forceUnlockAsync() {
        RFuture<Boolean> unlockFuture = lock.forceUnlockAsync();
        unlockFuture.whenComplete((unlockStatus, e) -> this.cancelExpirationRenewal());
        return unlockFuture;
    }

    @Override
    public RFuture<Void> unlockAsync() {
        RFuture<Void> unlockFuture = lock.unlockAsync();
        unlockFuture.whenComplete((unlockStatus, e) -> this.cancelExpirationRenewal());
        return unlockFuture;
    }

    @Override
    public RFuture<Void> unlockAsync(long threadId) {
        RFuture<Void> unlockFuture = lock.unlockAsync(threadId);
        unlockFuture.whenComplete((unlockStatus, e) -> this.cancelExpirationRenewal(threadId));
        return unlockFuture;
    }

    @Override
    public RFuture<Boolean> tryLockAsync() {
        return lock.tryLockAsync();
    }

    @Override
    public RFuture<Void> lockAsync() {
        return lock.lockAsync();
    }

    @Override
    public RFuture<Void> lockAsync(long threadId) {
        return lock.lockAsync(threadId);
    }

    @Override
    public RFuture<Void> lockAsync(long leaseTime, TimeUnit unit) {
        RFuture<Void> lockFuture = lock.lockAsync(leaseTime, unit);
        lockFuture.whenComplete((unlockStatus, e) -> {
            if (Objects.nonNull(e)) {
                return;
            }
            if (leaseTime > -1) {
                this.scheduleExpirationRenewal(leaseTime);
            }
        });
        return lockFuture;
    }

    @Override
    public RFuture<Void> lockAsync(long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Void> lockFuture = lock.lockAsync(leaseTime, unit, threadId);
        lockFuture.whenComplete((unlockStatus, e) -> {
            if (Objects.nonNull(e)) {
                return;
            }
            if (leaseTime > -1) {
                this.scheduleExpirationRenewal(leaseTime, threadId);
            }
        });
        return lockFuture;
    }

    @Override
    public RFuture<Boolean> tryLockAsync(long threadId) {
        return lock.tryLockAsync(threadId);
    }

    @Override
    public RFuture<Boolean> tryLockAsync(long waitTime, TimeUnit unit) {
        return lock.tryLockAsync(waitTime, unit);
    }

    @Override
    public RFuture<Boolean> tryLockAsync(long waitTime, long leaseTime, TimeUnit unit) {
        RFuture<Boolean> lockFuture = lock.tryLockAsync(waitTime, leaseTime, unit);
        lockFuture.whenComplete((unlockStatus, e) -> {
            if (Objects.nonNull(e)) {
                return;
            }
            if (leaseTime > -1) {
                this.scheduleExpirationRenewal(leaseTime);
            }
        });
        return lockFuture;
    }

    @Override
    public RFuture<Boolean> tryLockAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Boolean> lockFuture = lock.tryLockAsync(waitTime, leaseTime, unit, threadId);
        lockFuture.whenComplete((unlockStatus, e) -> {
            if (Objects.nonNull(e)) {
                return;
            }
            if (leaseTime > -1) {
                this.scheduleExpirationRenewal(leaseTime, threadId);
            }
        });
        return lockFuture;
    }

    @Override
    public RFuture<Integer> getHoldCountAsync() {
        return lock.getHoldCountAsync();
    }

    @Override
    public RFuture<Boolean> isLockedAsync() {
        return lock.isLockedAsync();
    }

    @Override
    public RFuture<Long> remainTimeToLiveAsync() {
        return lock.remainTimeToLiveAsync();
    }

}

```

有三个注意点：首先，这个装饰类需要放在 `org.redisson` 只有这样才能使用 `proteced void scheduleExpirationRenewal()` 和 `protected void cancelExpirationRenewal()` 方法。其次，续期依旧是按照 `internalLockLeaseTime` /3 间隔触发，但因为是申请锁完结之后的续期，所以此时的 `internalLockLeaseTime` 为第一次申请锁的 `leaseTime` 。第三，因 Redisson 版本不一样，可能不会有 RedissonBaseLock 基类，你可以升级版本后使用装饰类。