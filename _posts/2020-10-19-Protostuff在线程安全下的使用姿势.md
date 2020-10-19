---
layout:     post
title:      Protostuff在线程安全下的使用姿势
subtitle:   异常【Buffer previously used and had not been reset】的解决办法
date:       2020-10-19
author:     dengcaoyu
header-img: https://i.loli.net/2020/03/20/31dqoJGBkFOg9DA.jpg
catalog: true
tags:
    - Protostuff
---


## 需求背景
公司的push业务，有一些需要对对象序列化缓存到redis的场景。通过实现Serializable接口，将对象通过流的方式写入缓存。这种方式存在序列化后的码流太大、性能低等问题。对此，我将序列化框架改用Protostuff，代码编写、测试、发布上线，一切本以为朝着正确的方向前进者。

第二天来到公司，我就被同事告知：昨天的push失败了。通过查看日志，发现这样一个异常`Buffer previously used and had not been reset`。我反复检查我的代码，又在网上查询了各种案例，都没有遇到这种异常。

**以下是有异常的代码**
```java
package com.blackunique.bigdata.recommend.util;

import cn.hutool.core.util.ObjectUtil;
import io.protostuff.LinkedBuffer;
import io.protostuff.ProtostuffIOUtil;
import io.protostuff.Schema;
import io.protostuff.runtime.RuntimeSchema;
import lombok.NonNull;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Protostuff序列化/反序列化工具类
 *
 * @author Sharden
 * @create 2020-10-15 14:00
 */
public class ProtostuffUtils {

    /**
     * 缓存Schema
     */
    private static final Map<Class<?>, Schema<?>> SCHEMA_CACHE = new ConcurrentHashMap<>();
    /**
     * 避免每次序列化都重新申请Buffer空间
     */
    private static LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);

    /**
     * 序列化方法，把指定对象序列化成字节数组
     *
     * @param obj
     * @param <T>
     * @return
     */
    @SuppressWarnings("unchecked")
    public static <T> byte[] serialize(@NonNull T obj) {
        Class<T> cls = (Class<T>) obj.getClass();
        Schema<T> schema = getSchema(cls);
        byte[] data = null;
        try {
            data = ProtostuffIOUtil.toByteArray(obj, schema, buffer);
        } catch (Exception e) {
            throw new RuntimeException("序列化(" + obj.getClass() + ")对象(" + obj + ")发生异常!", e);
        } finally {
            buffer.clear();
        }
        return data;
    }

    /**
     * 反序列化方法，将字节数组反序列化成指定Class类型
     *
     * @param paramByteArray
     * @param cls
     * @param <T>
     * @return
     */
    public static <T> T deserialize(@NonNull byte[] paramByteArray, @NonNull Class<T> cls) {
        Schema<T> schema = getSchema(cls);
        T obj = schema.newMessage();
        ProtostuffIOUtil.mergeFrom(paramByteArray, obj, schema);
        return obj;
    }

    private static <T> Schema<T> getSchema(Class<T> cls) {
        Schema<T> schema = (Schema<T>) SCHEMA_CACHE.get(cls);
        if (ObjectUtil.isEmpty(schema)) {
            // schema通过RuntimeSchema进行懒创建并缓存
            schema = RuntimeSchema.getSchema(cls);
            if (ObjectUtil.isNotEmpty(schema)) {
                SCHEMA_CACHE.put(cls, schema);
            }
        }
        return schema;
    }
}

```

## 解决思路
通过看Protostuff的源码，发现这个异常是在下面这段代码里抛出的。咦，为啥`buffer.start != buffer.offset`呢？我明明在每次序列化完一个对象后，都调用`buffer.clear()`方法重置了start和offset的值啊？后来我又模拟线上的异常数据的序列化过程，通过debug打断点追踪，这个异常却没有复现...
```java
public static <T> byte[] toByteArray(T message, io.protostuff.Schema<T> schema, LinkedBuffer buffer) {
        if (buffer.start != buffer.offset) {
            throw new IllegalArgumentException("Buffer previously used and had not been reset.");
        } else {
            ProtostuffOutput output = new ProtostuffOutput(buffer);

            try {
                schema.writeTo(output, message);
            } catch (IOException var5) {
                throw new RuntimeException("Serializing to a byte array threw an IOException (should never happen).", var5);
            }

            return output.toByteArray();
        }
    }
```

正当我一筹莫展时，我发现这个LinkedBuffer对象居然是全局变量！等等，push业务是多线程跑的啊，那也就意味着，可能同时有多个线程都执行到`data = ProtostuffIOUtil.toByteArray(obj, schema, buffer)`这段代码。因为线程的执行顺序不确定，完全有可能在我们执行LinkedBuffer的clear方法之前，LinkedBuffer又被另一个线程使用了！在执行判断时，发现两个值不一致，抛出异常。

**线程安全的Protostuff工具类代码**
```java
package com.blackunique.bigdata.recommend.util;

import cn.hutool.core.util.ObjectUtil;
import io.protostuff.LinkedBuffer;
import io.protostuff.ProtostuffIOUtil;
import io.protostuff.Schema;
import io.protostuff.runtime.RuntimeSchema;
import lombok.NonNull;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Protostuff序列化/反序列化工具类
 *
 * @author Sharden
 * @create 2020-10-15 14:00
 */
public class ProtostuffUtils {

    /**
     * 缓存Schema
     */
    private static final Map<Class<?>, Schema<?>> SCHEMA_CACHE = new ConcurrentHashMap<>();

    /**
     * 序列化方法，把指定对象序列化成字节数组
     *
     * @param obj
     * @param <T>
     * @return
     */
    @SuppressWarnings("unchecked")
    public static <T> byte[] serialize(@NonNull T obj) {
        Class<T> cls = (Class<T>) obj.getClass();
        Schema<T> schema = getSchema(cls);
        byte[] data = null;
        try {
            LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
            data = ProtostuffIOUtil.toByteArray(obj, schema, buffer);
        } catch (Exception e) {
            throw new RuntimeException("序列化(" + obj.getClass() + ")对象(" + obj + ")发生异常!", e);
        }
        return data;
    }

    /**
     * 反序列化方法，将字节数组反序列化成指定Class类型
     *
     * @param paramByteArray
     * @param cls
     * @param <T>
     * @return
     */
    public static <T> T deserialize(@NonNull byte[] paramByteArray, @NonNull Class<T> cls) {
        Schema<T> schema = getSchema(cls);
        T obj = schema.newMessage();
        ProtostuffIOUtil.mergeFrom(paramByteArray, obj, schema);
        return obj;
    }

    private static <T> Schema<T> getSchema(Class<T> cls) {
        Schema<T> schema = (Schema<T>) SCHEMA_CACHE.get(cls);
        if (ObjectUtil.isEmpty(schema)) {
            synchronized (SCHEMA_CACHE) {
                schema = (Schema<T>) SCHEMA_CACHE.get(cls);
                if (ObjectUtil.isEmpty(schema)) {
                    // schema通过RuntimeSchema进行懒创建并缓存
                    schema = RuntimeSchema.getSchema(cls);
                    if (ObjectUtil.isNotEmpty(schema)) {
                        SCHEMA_CACHE.put(cls, schema);
                    }
                }
            }
        }
        return schema;
    }
}

```

## 总结
在使用多线程时，一定要考虑在并发环境下的线程安全问题。特别是使用全局变量，一定要慎之又慎，能不用就尽量不用，最好都用局部变量代替。
