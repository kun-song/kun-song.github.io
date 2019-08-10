---
title: Troubleshooting | MongoDB 安全认证的小坑
date: 2018-11-20 08:26:20
tags:
  - MongoDB
categories: 数据库
---

最近上线遇到一个蛋疼的问题，代码中需要对 `admin` 进行鉴权，在测试环境没有任何问题，但一上灰度就报错，报错信息也有点误导人，提示当前用户名、密码无法通过校验，本来以为是配置错误，后来发现同样的用户名、密码可以在 cli 鉴权通过，这就比较奇怪了。

<!-- more -->

stackoverflow 上有人说是 Spring Data MongoDB 版本过低导致的，升级下就 OK 了，我想这要么是 bug，要么是啥新特性，仔细比较了测试和灰度的数据库配置，发现所用用户的认证机制不同。

原来 MongoDB 3.0 之前使用 MONGODB-CR 机制进行认证，MongoDB 3.0 新增了 SCARAM-SHA-1，并将其作为默认的认证机制，因此添加用户时若无明确指定，认证机制都是 SCARAM-SHA-1。

而 Spring Data MongoDB 1.7.1 才 [兼容 SCARAM-SHA-1](https://docs.spring.io/spring-data/mongodb/docs/current/changelog.txt)，我们用的比这还老，所以就悲剧了。

前面说是版本过低导致的，这个说法对也不对，说他对是因为升级的确能解决该问题，说他不对是因为版本并非主要原因，不升级版本也能解决该问题。

很多客户端（如 robomongo）是兼容这两种认证机制的，所以平时使用时没有出现类似认证失败的问题。
