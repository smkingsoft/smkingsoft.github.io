# pglogical简介 #
                                                        翻译：神秘客 2017.01.06
<a href="https://2ndquadrant.com/en/resources/pglogical/">英文原文</a>

**pglogical是PostgreSQL的下一代复制系统**

pglogical是一个逻辑复制系统，可替代PostgreSQL物理复制。采用PostgreSQL扩展实现，具备完整的功能，无需额外的触发器或外部程序。pglogical采用发布/订阅模型，可以高效地且有选择地实现数据复制。

pglogical逻辑流复制允许数据库管理员执行以下功能：

* **迁移和升级--实现PostgreSQL的零停机迁移和升级**

<img src="https://2ndquadrant.com/media/photologue/photos/cache/Pglogical%20illustration%20A_RUmFgQL_display.png" alt="迁移和升级"></img>

* **聚合--将每个分片数据库的变更同步到数据仓库中**

<img src="https://2ndquadrant.com/media/photologue/photos/cache/Pglogical%20illustration%20C_xcLj2sT_display.png" alt="聚合"></img>

* **扩展--将所有节点或选定的数据库表数据复制到集群中的其他节点**

<img src="https://2ndquadrant.com/media/photologue/photos/cache/Pglogical%20illustration%20B_rUW8jhQ_display.png" alt="扩展"></img>

* **集成--将数据库变化实时反馈至其他系统**

<img src="https://2ndquadrant.com/media/photologue/photos/cache/Pglogical%20illustration%20D_6lHyfOl_display.png" alt="集成"></img>

## 可用性
(**最新!** 2017年1月4日 发布pglogical 1.2.2版本)
pglogical支持PostgreSQL9.4-9.6版本，Debian和Red Hat系列（RHEL、CentOS和Fedora）可以通过2ndQuadrant的apt和yum仓库获取安装包进行安装。
<a href="https://2ndquadrant.com/en/resources/pglogical/pglogical-installation-instructions/">详细安装介绍</a>
<a href="https://2ndquadrant.com/en/resources/pglogical/pglogical-docs/">详细文档</a>
<a href="http://2ndquadrant.com/en/resources/pglogical/release-notes/">发行说明</a> 
<a href="https://github.com/2ndQuadrant/pglogical">github源码获取</a>
pglogical基于PostgreSQL许可协议完全开源，即将合入PostgreSQL10版本核心功能。在pglogical被完全集成到PostgreSQL核心功能之前，由2ndQuadrant提供技术支持。

## 性能
使用pgBench工具，初步内部测试表明：pglogical在OLTP环境中，相对于其他复制软件（如slony、londiste3等），事务吞吐量增加了5倍。

<img src="https://2ndquadrant.com/media/photologue/photos/cache/pglogical_performance1_display.png"></img>

## 优势

* 无触发器，减少了发布端的写入负载；
* 不需要重新执行SQL，降低了订阅端的开销和延迟；
* 订阅端处于普通状态，即非热备状态，用户可以操作普通表以及temp、unlogged等属性；
* 取消查询不影响复制过程；
* 订阅端可以使用不同的用户、安全规则、索引以及参数设置；
* 采用复制集方式，支持单库或部分表复制；
* 支持异构设备(如windows、linux等)以及PostgreSQL不同版本之间的复制，从而达到零停机或者短暂停机升级；
* 支持多对一的变更汇聚复制。

## 工作机制

pglogical基于逻辑解码特性实现，该特性由2ndQuadrant公司开发，从PostgreSQL9.4版本提供支持。pglogical在PostgreSQL9.6版本中，执行速度更快且开销更低。

pglogical大量复用了双向复制<a href="https://2ndquadrant.com/en/resources/bdr/">BDR(bi-directional replication)</a>中经过实验和测试的技术，依赖于BDR所提供的部分特性：

* 逻辑解码
* 复制槽
* 后台静态进程
* 复制起点
* 事务提交时间戳

## 非BDR替代品

pglogical不提供BDR中实现的多主复制、序列复制以及DDL复制功能（译注：序列、DDL复制可有限支持，详见操作说明文档），仅结合BDR功能和使用经验，提供一种更简便易用的单向复制方案，从而具备更广阔的使用场景。BDR设计用于网状结构下的多向多主复制，很难适用于单主单向复制场景。

## 未来目标

pglogical采用模块化开发，将提供以下特性用例：

* 作为完整多主复制BDR的更新基础；
* 支持级联逻辑复制和复杂的复制拓扑；
* 订阅端应用时，通过触发器进行数据的提取和转换；
* 作为星形模型、列存储(cstore)的数据源；
* 选择性行级复制，例如：仅将客户各自的数据发送给各自的服务器；
* 从输出插件中流化json、xml等格式；
* 采用其他项目的输出插件替代基于触发器的数据采集
* 提供可扩展的，文本化的协议，适配多个不同客户端

## 技术支持
pglogical由2ndQuadrant开发并开源，对2ndQuadrant的支持客户提供完整技术支持。2ndQuadrant提供咨询服务，帮助应用开发、复制冲突设计、集群设计、部署、功能开发和性能分析。
