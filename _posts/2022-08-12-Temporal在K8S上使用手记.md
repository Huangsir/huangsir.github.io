---
layout: post
title: "Temporal在K8S上使用手记"
date: "2022-08-12"
categories: [Temporal]
---

#### 在 K8S 上部署Temporal

关于如何在 Production 环境部署 Temporal，官方有一份专门的[文档说明](https://docs.temporal.io/server/production-deployment/)

同时提供了一个[Helm Chart](https://github.com/temporalio/helm-charts)

Helm Chart 的 README 里提及这个 Helm Chart 并不建议直接用于生产环境，在[Community](https://community.temporal.io/t/kubernetes-deployment-for-production/1682/2)的问答里也有类似的回复，里面还提到，建议使用 `helm template` 然后再用*Anther method？*去往 K8S 部署。这个 Anther method 究竟是什么工具暂时也没想到

直接在 Rancher 上添加这个 helm chart 是无法使用的，会报错
```
Error: An error occurred while checking for chart dependencies. You may need to run 'helm dependency build' to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: cassandra, prometheus, elasticsearch, grafana
```

本地把项目 pull 下来然后运行 `helm dependency update` 之后倒是可以使用，但是总不能在本地发 production，不知道国外的小哥哥是不是不用 Rancher 这样的 UI 工具管理 K8S

翻来翻去，最终比较少的改动 Chart 的发布方案就是
1. 魔改 Chart.yaml 把 dependencies 全部注释掉，整个 Chart 就可以在 Rancher 上直接使用了
2. 用 `helm template` 生成 manifest，push 到自己的仓库里再去 Rancher 上发，但这个命令已经生成了最终执行的 manifest，没法再在 Rancher 启动时替换 values，就需要在其他地方去配置程序用的 Config

#### 平滑升级 

看了一下[Community](https://community.temporal.io/t/temporal-sql-tool-no-longer-in-the-temporalio-server-docker-image-should-we-use-temporalio-admin-tools-instead/2547/13)的讨论，emmmm...，基本也就是这个办法吧，好彩 Temporal Server 本身是向下兼容的

1. 先把 Helm 最新到最新版本
2. Deploy 一个 App 只包含 admintools
3. 然后用这个 admintools 更新表结构
4. 更新 Temporal Server 到最新版

```yaml
---
  web:
    enabled: false
  server:
    enabled: false
  schema:
    setup:
      enabled: false
    update:
      enabled: false
  prometheus:
    enabled: false
  grafana:
    enabled: false
  elasticsearch:
    enabled: false
  cassandra:
    enabled: false
  mysql:
    enabled: false
  admintools:
    enabled: true
```

如何执行 migration 可以在 Helm Chart 的 README 里找到

### MySQL 8

MySQL 的 migration schema 写的是 v57，我猜大概是 MySQL5.7 的意思，线上实际用的是 MySQL 8，这个 migration 实际在 MySQL 8 上是无法直接跑通的（好像 MySQL 5.7 也不行），会报错

同时 migration 不具有原子性，无法回滚，也就是说如果 migration create 2 个 table，第一个成功了，第二个失败了，这时第一个 table 会留在 DB里，重新运行 migration 会报第一个 table 已存在，无法继续执行，这里有点坑，如果是线上更新表出现问题，修复起来比较麻烦

最好是先在本地测试好，把跑不动的 SQL 魔改一下，再去线上执行

目前已知的错误是 `cluster_membership` 的建表语句 `TIMESTAMP DEFAULT '1970-01-01 00:00:01'` 会报 `Invalid default value` 错误

可以改成 `TIMESTAMP DEFAULT '0000-00-00 00:00:00'`，并在运行的时候用 `temporal-sql-tool --ca sql_mode=""` 修改运行的 SQL_MODE

当然，改成 `TIMESTAMP DEFAULT '0000-00-00 00:00:00'` 会不会引起程序运行的逻辑问题这个就没有验证了，最好的处理就是不要用 MySQL8，直接用 MySQL5.7

`A feel moments later...`

MySQL 5.7 也是不行的，实测发现是这样的

```sql
CREATE TABLE cluster_membership
(
    membership_partition INT NOT NULL,
    host_id              BINARY(16) NOT NULL,
    rpc_address          VARCHAR(15) NOT NULL,
    rpc_port             SMALLINT NOT NULL,
    role                 TINYINT NOT NULL,
    session_start        TIMESTAMP DEFAULT '0000-00-00 00:00:00', -- 需要修改 SQL_MODE 去掉 NO_ZERO_DATE 否则会报错
    last_heartbeat       TIMESTAMP DEFAULT '1970-01-01 00:00:01+08.00',  -- 可以直接运行
    record_expiry        TIMESTAMP DEFAULT '1970-01-01 00:00:01+00.00',  -- 可以直接运行
    INDEX (role, host_id),
    INDEX (role, last_heartbeat),
    INDEX (rpc_address, role),
    INDEX (last_heartbeat),
    INDEX (record_expiry),
    PRIMARY KEY (membership_partition, host_id)
);
```

建表之后用 `show create table cluster_membership` 看发现，这几个字段初始化的结果是一样的

```sql
| cluster_membership | CREATE TABLE `cluster_membership` (
  `membership_partition` int(11) NOT NULL,
  `host_id` binary(16) NOT NULL,
  `rpc_address` varchar(15) NOT NULL,
  `rpc_port` smallint(6) NOT NULL,
  `role` tinyint(4) NOT NULL,
  `session_start` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `last_heartbeat` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `record_expiry` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  PRIMARY KEY (`membership_partition`,`host_id`),
  KEY `role` (`role`,`host_id`),
  KEY `role_2` (`role`,`last_heartbeat`),
  KEY `rpc_address` (`rpc_address`,`role`),
  KEY `last_heartbeat` (`last_heartbeat`),
  KEY `record_expiry` (`record_expiry`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
```

