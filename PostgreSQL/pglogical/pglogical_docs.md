# pglogical参考文档
                                                        翻译：神秘客 2017.01.08
<a href="https://2ndquadrant.com/en/resources/pglogical/pglogical-docs/">英文原文</a>

PostgreSQL的插件pglogical使用发布/订阅模式，为数据库提供逻辑流复制功能，pglogical基于<a href="http://2ndquadrant.com/BDR">BDR项目</a>的部分技术完成。

复用Slony技术中的下列术语来描述各节点之间的数据流：

* 节点-PostgreSQL数据库实例
* 发布端和订阅端-节点所承担的角色
* 复制集-需要复制的表集合

pglogical利用了最新PostgreSQL内核的新技术，对所使用的数据库版本有最低限制如下：

* 数据的发布和订阅节点必须为PostgreSQL9.4或更高版本。
* 复制源过滤和冲突检测需要PostgreSQL9.5或更高版本。

支持的使用场景：

* 主版本数据库之间的升级（节点满足上述版本限制）
* 数据库全库复制
* 利用复制集，选择性的复制部分表
* 从多个上游数据源聚集和合并数据

架构细节

* pglogical运行于每个数据库级别，而不是同物理复制一样运行于整个服务器实例级别；
* 一个发布端可以支持多个订阅端，且不会引起额外的磁盘写入开销；
* 一个订阅端可以合并来自多个源的变更，并且可以自动检测变更冲突，配置冲突解决方案（仅支持部分冲突解决，而不是多主复制的所有方面）；
* 通过变更集转发的方式实现级联复制；

## 1. 安装要求

发布端和订阅端必须运行PostgreSQL9.4或更高版本，才可以使用pglogical组件。

pglogical_output插件在发布端和订阅端都必须安装，但无需执行 ``` CREATE EXTENSION ``` 命令进行实际安装，仅需在PostgreSQL运行目录下存在即可。

pglogical插件在发布端和订阅端都必须安装，且必须执行 ``` CREATE EXTENSION pglogical ``` 命令在两端完成实际安装。

发布端和订阅端的表必须具有相同的名称且归属于相同的schema，未来版本可以提供映射功能；

发布端和订阅端的表必须具有相同的字段和字段数据类型，订阅端的约束、非空(NOT NULL)约束等必须与发布端相同或比其更弱（更宽松）；

表必须有相同的主键，不建议增加除主键之外的唯一(UNIQUE)约束（详见[下文](#Only one unique index/constraint/PK)）；

其他额外要求详见下文“[限制和约束](#Limitations and Restrictions)”章节；

## 2. 用法

本节介绍复制插件pglogical的基本用法

### 2.1 快速安装

首先PostgreSQL服务器已经被正确配置为开启逻辑复制：
```
wal_level = 'logical'
max_worker_processes = 10   # 发布端节点的每个节点均需要配置合适的值
                            # 订阅端节点的每个节点均需要配置合适的值
max_replication_slots = 10  # 发布端节点的每个节点均需要配置合适的值
max_wal_senders = 10        # 发布端节点的每个节点均需要配置合适的值
shared_preload_libraries = 'pglogical'  #加载插件动态库，若需加载多个插件以逗号分隔
```
PostgreSQL9.5或更高版本（9.4不支持）可以附加选项至配置文件 *postgresql.conf* 中，设置冲突处理原则：以首次更新为准还是末次更新为准（详见“[冲突](#Conflicts)”章节）。
```
track_commit_timestamp = on # 设置冲突处理原则是首次更新或末次更新为准
                            # 该选项仅支持PostgreSQL9.5或更高版本
```
*pg_hba.conf* 配置成运行具有复制权限的用户从本地发起复制连接。

其次 *pglogical* 已经安装在所有节点：
```
CREATE EXTENSION pglogical;
```
然后创建发布端节点：
```
SELECT pglogical.create_node(
    node_name := 'provider1',
    dsn := 'host=providerhost port=5432 dbname=db'
);
```
将所有位于 *public schema* 的表加入缺省的复制集：
```
SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);
```
当然也可创建其他复制集，并加入需要复制的表（详见“[复制集](#Replication Sets)”章节）。
通常建议在创建订阅端之前创建好复制集，以便于在初始化复制设置的单一初始事务中同步所有的表。当然为了方便控制，对于较大的数据库，也可以在创建订阅端后，增量地创建复制集。

当发布端完成安装后，订阅端即可进行订阅，先创建订阅节点：
```
SELECT pglogical.create_node(
    node_name := 'subscriber1',
    dsn := 'host=thishost port=5432 dbname=db'
);
```
最后在订阅节点创建订阅，在后台启动同步和复制进程。
```
SELECT pglogical.create_subscription(
    subscription_name := 'subscription1',
    provider_dsn := 'host=providerhost port=5432 dbname=db'
);
```

### 2.2 节点管理

节点可以通过SQL命令进行动态的增加和删除。

* 创建节点：``` pglogical.create_node(node_name name, dsn text) ```
    参数说明：
    - *node_name* 新节点名称，一个数据库只能有一个节点；
    - *dsn* 数据源名称（译注：*Data Source Name*缩写），对于发布端节点，应该可以从外部连接；
* 删除节点：``` pglogical.drop_node(node_name name, ifexists bool) ```
    参数说明：
    - *node_name* 待删除的现有节点名称；
    - *ifexists* 设置为 *true*，则订阅不存在时不报错，缺省值： *false*；
* 增加接口：``` pglogical.alter_node_add_interface(node_name name, interface_name name, dsn text) ```
使用 *create_node* 命令创建节点时，会同时使用 *dsn* 指定的数据源，创建与节点同名的接口。本命令可以在节点上增加不同数据源名称的附加接口。
    参数说明：
    - *node_name* 现有节点名称；
    - *interface_name* 待增加的新接口名称；
    - *dsn* 新接口的数据源名称；
* 删除接口：``` pglogical.alter_node_drop_interface(node_name name, interface_name name) ```
从节点现有接口中删除一个接口
    参数说明：
    - *node_name* 现有节点名称；
    - *interface_name* 节点中待删除的现有接口名称

### 2.3 订阅管理

* 创建订阅：``` pglogical.create_subscription(subscription_name name, provider_dsn text, replication_sets text[], synchronize_structure boolean, synchronize_data boolean, forward_origins text[]) ```
从当前节点创建一个到发布节点的订阅，命令为非阻塞方式，仅仅启动操作。
    参数说明：
    - *subscription_name* 订阅名称，必须保证唯一；
    - *provider_dsn* 发布端数据源名称；
    - *replication_sets* 复制集组，缺省值" *{default,default_insert_only}* "，这些组必须存在；
    - *synchronize_structure* 是否同步结构，缺省值： *false*；
    - *synchronize_data* 是否通过数据，缺省值： *true*；
    - *forward_origins* 需要转发的源名称组，当前仅支持"*{}*"和"*{all}*"两个值，"*{}*"表示不转发任何非发布端的变更,"*{all}*"表示不区分来源复制所有变更；
* 删除订阅：``` pglogical.drop_subscription(subscription_name name, ifexists bool) ```
断开与发布端的连接，并从目录(catalog)中删除
    参数说明：
    - *subscription_name* 待删除的订阅名称；
    - *ifexists* 设置为 *true*，则订阅不存在时不报错，缺省值： *false*；
* 禁用订阅：``` pglogical.alter_subscription_disable(subscription_name name, immediate bool) ```
停止订阅，并断开与发布端的连接
    参数说明：
    - *subscription_name* 待禁用的订阅名称
    - *immediate* 设置为 *true*,表示立即停止订阅，否则等当前事务结束后才停止，缺省值： *false*；
* 启用订阅：``` pglogical.alter_subscription_enable(subscription_name name, immediate bool) ```
启用被禁用的订阅。
    参数说明：
    - *subscription_name* 待启用的订阅名称
    - *immediate* 设置为 *true*,表示立即启用订阅，否则等当前事务结束后才启用，缺省值： *false*;
* 切换接口：``` pglogical.alter_subscription_interface(subscription_name name, interface_name name) ```
切换订阅用不同的接口连接发布节点。
    - *subscription_name* 现有订阅名称
    - *interface_name* 当前发布节点上的接口名称
* 同步订阅：``` pglogical.alter_subscription_synchronize(subscription_name name, truncate bool) ```
在单个操作中同步所有复制集中未同步的表，这些表会被逐个的复制和同步，命令为非阻塞方式，仅仅启动操作。
    参数说明：
    - *subscription_name* 现有订阅名称
    - *truncate* 设置为 *true*,表示复制前先截断表，缺省值： *false*;
* 重新同步：``` pglogical.alter_subscription_resynchronize_table(subscription_name name, relation regclass) ```
重新同步一个存在的表
**警告：本功能同步前会先截断表**
    参数说明：
    - *subscription_name* 现有订阅名称
    - *relation* 待重新同步的表名，此参数可选
* 订阅状态显示：``` pglogical.show_subscription_status(subscription_name name) ```
显示订阅的基本信息和状态
    参数说明：
    - *subscription_name* 订阅名称，此参数可选，未提供参数时，显示本节点上所有订阅的信息；
* 表状态显示：``` pglogical.show_subscription_table(subscription_name name, relation regclass) ```
显示某个表的同步状态
    参数说明：
    - *subscription_name* 现有订阅名称
    - *relation* 待显示状态的表名，此参数可选
* 增加复制集：``` pglogical.alter_subscription_add_replication_set(subscription_name name, replication_set name) ```
增加一个复制集至订阅端，仅仅是激活事件，不进行同步。
    参数说明：
    - *subscription_name* 现有订阅名称；
    - *replication_set* 待增加的复制集名称；
* 删除复制集：``` pglogical.alter_subscription_remove_replication_set(subscription_name name, replication_set name) ```
从订阅端删除一个复制集
    参数说明：
    - *subscription_name* 现有订阅名称；
    - *replication_set* 待删除的复制集名称；

### 2.4 <span id="Replication Sets">复制集</span>

复制集提供了一种控制机制，控制数据库中需要被复制的表，以及这些表上需要被复制的操作。每个复制集都可以单独指定具体的操作，如：*insert*,*update*,*delete*,*truncate*等。一个表可在多个复制集中存在；每个订阅端可以订阅多个复制集。被复制的表和操作是这些表所在复制集的并集，不在复制集中的表不会被复制。
目前预置了两个缺省复制集，名称为"*default*"，"*default_insert_only*"。"*default*"复制集定义为复制集合中表的所有更改；"*default_insert_only*"复制集仅复制insert操作，用于没有主键的表。

下列函数用于管理复制集信息：

* 创建复制集：``` pglogical.create_replication_set(set_name name, replicate_insert bool, replicate_update bool, replicate_delete bool, replicate_truncate bool) ```
    参数说明：
    - *set_name* 复制集名称，必须唯一;
    - *replicate_insert* 是否复制insert操作，缺省值："*true*";
    - *replicate_update* 是否复制update操作，缺省值："*true*";
    - *replicate_delete* 是否复制delete操作，缺省值："*true*";
    - *replicate_truncate* 是否复制truncate操作，缺省值："*true*";
* 修改复制集参数： ``` pglogical.alter_replication_set(set_name name, replicate_inserts bool, replicate_updates bool, replicate_deletes bool, replicate_truncate bool) ```
修改已存在的复制集参数
    参数说明：
    - *set_name* 待修改的复制集名称；
    - *replicate_insert* 是否复制insert操作，缺省值："*true*";
    - *replicate_update* 是否复制update操作，缺省值："*true*";
    - *replicate_delete* 是否复制delete操作，缺省值："*true*";
    - *replicate_truncate* 是否复制truncate操作，缺省值："*true*";
* 删除复制集：``` pglogical.drop_replication_set(set_name text) ```
    参数说明：
    - *set_name* 待删除的复制集名称
* 增加单个表：``` pglogical.replication_set_add_table(set_name name, relation regclass, synchronize_data boolean) ```
增加一个需要复制的表到复制集
    参数说明：
    - *set_name* 现有复制集名称；
    - *relation* 待加入的表名称或者表OID；
    - *synchronize_data* 是否将表数据同步给所有订阅了此复制集的订阅端，缺省值："*false*"；
* 增加所有表：``` pglogical.replication_set_add_all_tables(set_name name, schema_names text[], synchronize_data boolean) ```
将指定schema组中所有表都加入复制集，仅针对当前存在表，将来新增的表不会被自动加入。若希望将来新增表自动加入合适的复制集，请参阅“[新表自动分配复制集](#Automatic Assignment of Replication Sets)”章节
    参数说明：
    - 
* 删除表：``` pglogical.replication_set_remove_table(set_name name, relation regclass) ```
    参数说明：
    - *set_name* 现有复制集名称
    - *relation* 待删除的表名或表OID
* 增加单个序列：``` pglogical.replication_set_add_sequence(set_name name, relation regclass, synchronize_data boolean) ```
    参数说明：
    - *set_name* 现有复制集名称；
    - *relation* 待加入的序列名称或序列OID；
    - *synchronize_data* 是否立即同步序列，缺省值："*false*"；
* 增加所有序列：``` pglogical.replication_set_add_all_sequences(set_name name, schema_names text[], synchronize_data boolean) ```
将指定schema组中所有序列都加入复制集，但仅针对当前存在的序列，将来新增的序列不会被自动加入。
    参数说明：
    - *set_name* 现有复制集名称；
    - *schema_names* 待加入的序列所属schema名称组
    - *synchronize_data* 是否立即同步序列，缺省值："*false*"；
* 删除序列：``` pglogical.replication_set_remove_sequence(set_name name, relation regclass) ```
    参数说明：
    - *set_name* 现有复制集名称；
    - *relation* 待删除的序列名称或者序列OID；

可以查询视图 *pglogical.tables*，获取表与复制集的归属信息。


#### 2.4.1 <span id="Automatic Assignment of Replication Sets">新表自动分配复制集</span>

通过事件触发器为新创建的表定义合适的复制集，例如：
```
CREATE OR REPLACE FUNCTION pglogical_assign_repset()
RETURNS event_trigger AS $$
DECLARE obj record;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        IF obj.object_type = 'table' THEN
            IF obj.schema_name = 'config' THEN
                PERFORM pglogical.replication_set_add_table('configuration', obj.objid);
            ELSIF NOT obj.in_extension THEN
                PERFORM pglogical.replication_set_add_table('default', obj.objid);
            END IF;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER pglogical_assign_repset_trg
    ON ddl_command_end
    WHEN TAG IN ('CREATE TABLE', 'CREATE TABLE AS')
    EXECUTE PROCEDURE pglogical_assign_repset();
```
上例将所有属于 *config Schema*的新增表放入 *configuration*复制集，而其他非扩展创建的新增表放入缺省复制集中。

### 2.5 附加功能

* 复制DDL：``` pglogical.replicate_ddl_command(command text, replication_sets text[]) ```
本地执行后，相关指令将发送至复制队列，同步给订阅了指定复制集的订阅端执行。
    参数说明：
    - *command* DDL命令
    - *replication_sets* 与DDL命令相关的复制集组，缺省值："*{ddl_sql}*"
* 推送序列: ``` pglogical.synchronize_sequence(relation regclass) ```
推送序列至所有订阅端，与普通订阅和表同步功能不同的是，此功能应该在发布端执行。该命令执行后，一旦所有订阅端应用了此事务，这个序列在订阅端是被强制更新的。（复制集过滤依然适用）
    参数说明：
    - *relation* 待推送的序列名称，此参数可选

## 3 <span id="Conflicts">冲突</span>

本节说明适用于订阅节点存在多个发行端，或者订阅端本地存在写入的时候，传入更改发生冲突的情况。这些冲突可以自动检测，并根据配置采取解决方案。

通过设置配置项 *pglogical.conflict_resolution* 完成冲突解决方案的配置，其取值如下：

* *error* -检测到冲突，且希望手动操作解决时，复制将在错误处停止；
* *apply_remote* -当变更与本地数据发生冲突时，以源端数据为准，此为 **缺省值**；
* *keep_local* -保留本地数据变更，忽略来着远端节点的变更冲突；
* *last_update_wins* -保留具有最新提交时间戳的变更，不管其来源是本地还是远端；
* *first_update_wins* -保留具有最早时间戳的变更，不管其来源是本地还是远端；

当禁用配置项 *track_commit_timestamp*时，仅可设置 *apply_remote*。因PostgreSQL9.4不支持配置项 *track_commit_timestamp*，所以也只能用缺省值 *apply_remote*。

## 4. <span id="Limitations and Restrictions">限制和约束</span>

### 4.1 超级用户

pglogical当前版本需要通过超级用户执行复制，未来为扩展为具有复制权限的用户。

### 4.2 ``` UNLOGGED ```和``` TEMPORARY ```

与物理流复制类似，设置了 *UNLOGGED* 和 *TEMPORARY* 属性的表不会被复制，也无法被复制。

### 4.3 一库一配置

复制多个数据库时，必须为每个库设置各自的发行端和订阅端。无法一次性在PostgreSQL安装过程中设置所有数据库的复制。

### 4.4 主键或副本标识

无主键或者有效副本标识（如：唯一约束）的表无法复制更新和删除操作。因为没有唯一标识，复制程序无法找到对应的更新/删除记录。

### 4.5 <span id="Only one unique index/constraint/PK">唯一索引/约束或主键</span>

配置多个上游复制源或者下游允许本地变更写入时，下游复制表中仅可存在一个唯一索引。因为冲突解决方案一次仅能使用一个索引，如果行满足主键约束，但是下游侧违反其他唯一约束时，则冲突行会报错，导致复制停止，直到下游表的冲突行被删除。

若下游仅从上游或其他地方获取写入，上游具备额外唯一约束更有利于复制。基本规则是下游约束不能比上游约束更严格。

### 4.6 DDL

pglogical不会自动进行DDL复制。管理DDL，保持发布端和订阅端数据库兼容是用户的责任。

### 4.7 无复制队列刷新

不支持冻结主节点上的事务并一直等待到所有待处理排队的xact从槽中重播。在将来的版本中添加此特性对上游为只读场景的支持。

因此变更表结构时必须要小心，防止发行端和订阅端同时变更表结构，导致订阅端的表与排队事务不兼容时，如果存在未复制的已提交事务，复制过程会停止。

管理员应该在变更 *schema*之前确保主服务器已经停止写入，或者使用 ``` pglogical.replicate_ddl_command ```函数对 *schema*进行排队，以便在复制的一致点处进行重放。

但是在多主复制的场景下，仅使用 ``` pglogical.replicate_ddl_command ```是不够的。因为订阅端可能在发布端提交 *schema*变更后，使用旧结构生成新的xact事务。用户必须保证在所有节点上停止写入，并确保所有的插槽在 *schema*更改前都已被捕获。

### 4.8 外键

复制过程中不强制保证外键约束，即使违反外键约束，订阅端也可以成功应用从发行端获取的变更。

### 4.9 截断

使用 ``` TRUNCATE ... CASCADE ```命令仅能在发行端完成 *CASCADE*选项。（可能需要PostgreSQL附加 ``` ON TRUNCATE CASCADE ```支持外键的特性，才能正确处理这个问题。）

不支持 ``` TRUNCATE ... RESTART IDENTITY ```命令，唯一标识(identity)重新初始化步骤不会被复制到副本中。

### 4.10 序列

复制集中的序列是周期性复制，而不是实时复制。使用动态缓冲区记录被复制的序列值，以便订阅端接收序列下个周期的值。这种方式能将订阅端所看到的序列 * last_value*落后问题最小化，但不能完全消除可能性。

在数据库有大事件发生比如导入数据后或在线升级期间，建议调用 *synchronize_sequence* 确保所有订阅端都有给定序列的最新值。

建议在多节点系统上的序列使用bigserial和bigint类型，因为较小的序列可能很快达到序列取值上限。

若希望发布端和订阅端使用独立的序列，则不要将序列加入复制集。创建步长大于等于节点数的序列，并在每个节点上设置不同的偏移量。在命令``` CREATE SEQUENCE ```或``` ALTER SEQUENCE ```中使用``` INCREMENT BY ```选项，通过``` setval(...) ```命令设置序列起点。

### 4.11 触发器

应用进程和初始化拷贝（译注：COPY）进程都使用 *session_replication_role*规则， 设置为 *replia*意味着``` ENABLE REPLICA ```和``` ENABLE ALWAYS ```触发器会被触发。

### 4.12 版本间的差异

pglogical可以跨PostgreSQL主版本复制，尽管可以正常工作，但是长时间的跨版本复制并不是设计初衷。跨版本复制的时候，容易发生发布端的有效变更在订阅端不可用的情况。

从旧版本到新版本的复制比较安全，因为PostgreSQL保持了稳定的向后兼容性，但仅有有限的向前兼容性。

不同小版本之间的复制没有差异。

### 4.13 不复制DDL

逻辑解码不直接解码目录(calalog)更改，因此新增表时，插件不能仅仅发送``` CREATE TABLE ```语句。

被解码的数据应用到另外一个PostgreSQL数据库之前，对应的表定义必须通过逻辑解码插件之外的手段保持同步。例如：

* 通过DDL逆分析事件触发器捕获DDL变更，将其写入要复制的表，并应用到另外一端；
* 通过DDL管理工具，在所有节点上同步DDL；

## 与BDR的差异

pglogical基于为BDR开发的某些技术，并与BDR共享了一些代码。设计比BDR更灵活，适用于单主单向复制、数据收集/合并、非网格多主机拓扑等等。

省略了BDR中的以下功能：

* 网状多主复制。有限地支持存在冲突解决方案的多主复制，必须分别单独添加相互复制的连接；
* 分布式序列。在每个节点上使用不同的序列偏移；
* DDL复制。用户必须自行保持表定义的一致性，pglogical提供队列功能帮助用户完成操作；
* 全局DDL锁定。pglogical不支持DDL复制，也无需全局DDL锁定。仅应用于表的DDL复制也会引入多主相互之间的复制问题。详见下一点；
* 全局状态一致性刷新。BDR的DDL锁定是所有节点防止新的xact被提交插入到队列从而刷新到对端节点的步骤之一。这样当表结构发生变更时，确保队列中没有xact存在，也不能被应用。pglogical没有实现此功能，因此不支持多主复制（一些节点之间相互复制），详见[限制和约束](#Limitations and Restrictions)

pglogical增加了一些新特性：

* 节点间的灵活连接，不限于类似BDR一样的网格拓扑，可以实现级联逻辑复制。
* 输出组件为松耦合，可以被其他项目复用。
* JSON输出格式，可以检测队列事务。

主要目的是为了提供一个简单便捷的基础，使用pglogical不需要PostgreSQL打补丁，pglogical是可插拔和可扩展的PostgreSQL插件。
