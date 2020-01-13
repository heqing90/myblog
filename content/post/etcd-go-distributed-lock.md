---
title: "Etcd client V3 分布式锁实现方式"
date: 2020-01-13T16:15:23+08:00
draft: true
tags:
    - "etcd"
    - "Golang"
    - "分布式锁"
categories:
    - "入门"
---

## 原理

1. 客户端以统一前缀（比如/mylock）创建全局唯一Key（client 1 -> /mylock/client-1-uuid, client 2 -> /mylock/client-2-uuid ）。

2. 为该Key绑定租约(lease)，并设定TTL，客户端需创建keepalive任务刷新TTL时间，避免死锁。

3. 客户端将上诉 *Key-Value* 写入ETCD，并获取锁前缀(/mylock)下所有键值，根据返回的 *reversion* 的大小判断自己能否持有锁。
    - 拥有最小reversion则表明可以持有锁。
    - 自己的reversion不是最小则等待直到上一个reversion的键值被删掉，然后持有该锁
4. 删除  *lock key*，则表明释放该锁

## 源码分析

1. ***session***
    - Keepalive

    ```go
        func (l *lessor) KeepAlive(ctx context.Context, id LeaseID) (<-chan *LeaseKeepAliveResponse, error) {
            ...
            go l.keepAliveCtxCloser(ctx, id, ka.donec)
            l.firstKeepAliveOnce.Do(func() {
                // 根据服务器回复的TTL时间判断租约是否存在
                go l.recvKeepAliveLoop()
                // 网络异常情况下，根据自己的Timer判断租约是否存在
                go l.deadlineLoop()
            })
            ...
        }
        func (l *lessor) recvKeepAliveLoop() (gerr error) {
            ...
            for {
                // 开启心跳发送任务
                stream, err := l.resetRecv()
                if err != nil {
                    ...
                } else {
                    // 根据服务端的响应刷新 lease TTL 时间（TTL/3 触发刷新）
                    l.recvKeepAlive(resp)
                }
            }
            ...
        }

        func (l *lessor) recvKeepAlive(resp *pb.LeaseKeepAliveResponse) {

            // send update to all channels
            // deadline 以接受到服务器响应的时间为基准再加上TTL， 会比服务端的TTL多一个网络延时
            nextKeepAlive := time.Now().Add((time.Duration(karesp.TTL) * time.Second) / 3.0)
            ka.deadline = time.Now().Add(time.Duration(karesp.TTL) * time.Second)

        }
    ```  

    - Session Done

    ```go
        // 租约失效时触发, 可监听该channel,在租约失效时做对应处理
        func (s *Session) Done() <-chan struct{} { return s.donec }
    ```

2. ***mutex***
    - 加锁

    ```go
        func (m *Mutex) Lock(ctx context.Context) error {
            resp, err := m.tryAcquire(ctx)
            if err != nil {
                return err
            }
            // 如果该客户端的key的create version 是最小的则持有该锁
            ownerKey := resp.Responses[1].GetResponseRange().Kvs
            if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {
                m.hdr = resp.Header
                return nil
            }
            client := m.s.Client()
            // 如果该锁被其他客服端持有，则等待该lockkey下拥有上一个reversion的键值被删除
            hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
            // release lock key if wait failed
            if werr != nil {
                m.Unlock(client.Ctx())
            } else {
                m.hdr = hdr
            }

            // make sure the session is not expired, and the owner key still exists.
            gresp, werr := client.Get(ctx, m.myKey)
            if werr != nil {
                m.Unlock(client.Ctx())
                return werr
            }

            if len(gresp.Kvs) == 0 { // is the session key lost?
                return ErrSessionExpired
            }
            m.hdr = gresp.Header

            return nil
        }

        func (m *Mutex) tryAcquire(ctx context.Context) (*v3.TxnResponse, error) {
            s := m.s
            client := m.s.Client()
            // 以参数 lockkey+leaseID 作为该客户端锁的全局唯一标识
            // 如果该锁不存在则创建，否则直接获取，取得该键值的create version 同时请求以lockkey为前缀的所有键值
            m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())
            cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)
            // put self in lock waiters via myKey; oldest waiter holds lock
            put := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))
            // reuse key in case this session already holds the lock
            get := v3.OpGet(m.myKey)
            // fetch current holder to complete uncontended path with only one RPC
            getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
            resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
            if err != nil {
                return nil, err
            }
            m.myRev = resp.Header.Revision
            if !resp.Succeeded {
                m.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision
            }
            return resp, nil
        }
    ```

    - 解锁

    ```go
        func (m *Mutex) Unlock(ctx context.Context) error {
            client := m.s.Client()
            // 直接删除该客户端对应锁的键值并将锁的本地reversion置为-1
            if _, err := client.Delete(ctx, m.myKey); err != nil {
                return err
            }
            m.myKey = "\x00"
            m.myRev = -1
            return nil
        }
    ```

## 小结

 1. 使用ETCD分布式锁 *concurrency.Mutex* 时，一定要加上 *concurrency.Session*, 避免程序或网络发生异常导致死锁

 2. 对于 *session.Done* 的触发机制要注意，本地deadline检测的依据是 timer > TTL + 网络延时，这时服务器可能早已删除了对应的Key。如果本地客户端在这个差异时间内依然持有锁并做一些操作的话，可能会让锁失效。
