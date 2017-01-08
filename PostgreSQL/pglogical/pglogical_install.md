# pglogical安装说明
                                                        翻译：神秘客 2017.01.07
<a href="https://2ndquadrant.com/en/resources/pglogical/pglogical-installation-instructions/">英文原文</a>

pglogical在Fedora、CentOS、RHEL等系统中可通过yum使用RPM包方式安装，在Debian和Ubuntu等系统中可通过apt使用DEB包方式安装。所有系统均可以通过源码方式安装(<a href="http://packages.2ndquadrant.com/pglogical/tarballs/">主站源码</a>，<a href="https://github.com/2ndQuadrant/pglogical">github源码</a>)，操作详见下面的具体章节

## YUM仓库
以下说明适用于Red Hat系列操作系统（RHEL、CentOS和Fedora）

### 预置条件
所有的RPM包，都依赖PGDG(PostgreSQL Global Development Group
)包和PostgreSQL发行包，依赖包可以从<a href="http://yum.postgresql.org/">http://yum.postgresql.org/</a>获取，不要使用Fedora和RHEL系统中自带的PostgreSQL版本。
若系统中未安装PostgreSQL，可以通过下面的方式获取安装包并安装

* 从<a href="http://yum.postgresql.org/repopackages.php">http://yum.postgresql.org/repopackages.php</a>选择合适的PGDG RPM包安装
* 安装PostgreSQL
    - **V9.4：**通过yum安装postgresql94-server postgresql94-contrib
    - **V9.5：**通过yum安装postgresql95-server postgresql95-contrib
    - **V9.6：**通过yum安装postgresql96-server postgresql96-contrib

### 安装
首先安装发行版本的RPM仓库

* Fedora 24和25：通过yum安装<a href="http://packages.2ndquadrant.com/pglogical/yum-repo-rpms/pglogical-fedora-1.0-2.noarch.rpm">http://packages.2ndquadrant.com/pglogical/yum-repo-rpms/pglogical-fedora-1.0-2.noarch.rpm</a>
* RHEL/CentOS 6和7：通过yum安装<a href="http://packages.2ndquadrant.com/pglogical/yum-repo-rpms/pglogical-fedora-1.0-2.noarch.rpm">http://packages.2ndquadrant.com/pglogical/yum-repo-rpms/pglogical-fedora-1.0-2.noarch.rpm</a>

然后安装与PostgreSQL版本相匹配的pglogical：

* **PostgreSQL 9.4：**通过yum安装postgresql94-pglogical
* **PostgreSQL 9.5：**通过yum安装postgresql95-pglogical
* **PostgreSQL 9.6：**通过yum安装postgresql96-pglogical

**软件包签名**
安装过程中，可能提示接受软件包签名的仓库GPG密钥:
```
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-2NDQ-PGLOGICAL
Importing GPG key 0xEB74CAFE:
Userid     : "pglogical repository key (2ndQuadrant) <packagers@2ndquadrant.com>"
Fingerprint: 0005 a7af 6208 3d86 d273 d5cb bc66 be8f eb74 cafe
Package    : pglogical-fedora-1.0-1.noarch (installed)
From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-2NDQ-PGLOGICAL
Is this ok [y/N]:
```
若提示与上文举例相匹配，请输入"y"并回车，接受此密钥。
**注：**此密钥由2ndQuadrant主软件包密钥签名，可依据主密钥进行验证。

## APT仓库
以下说明适用于Debian以及所有基于Debian的Linux版本（如：Ubuntu）

### 预置条件
首先按照下列步骤执行，加入2ndQuadrant软件仓库源

* 创建 */etc/apt/sources.list.d/2ndquadrant.list* 文件，记录2ndquadrant发行代号；
* 将下面命令中的"*jessie*"替换为系统中实际使用的发行代号（通过命令 *lsb_release -c* 获取）；
```
deb [arch=amd64] http://packages.2ndquadrant.com/pglogical/apt/ jessie-2ndquadrant main
```
* 增加仓库<a href="http://apt.postgresql.org/">http://apt.postgresql.org/</a>，仓库详细说明参见网站；
* 导入2ndQuadrant的apt仓库密钥，更新软件包列表，即可安装相关软件包；
```
wget --quiet -O - http://packages.2ndquadrant.com/pglogical/apt/AA7A6805.asc | sudo apt-key add
sudo apt-get update 
```

### 安装

然后安装与PostgreSQL版本相匹配的pglogical:

* **PostgreSQL 9.4：** ``` sudo apt-get install postgresql-9.4-pglogical ``` 
* **PostgreSQL 9.5：** ``` sudo apt-get install postgresql-9.5-pglogical ``` 
* **PostgreSQL 9.6：** ``` sudo apt-get install postgresql-9.6-pglogical ``` 

## 源码安装

pglogical源码方式与其他PostgreSQL扩展安装类似，使用PGXS构建。
首先确定PostgreSQL版本pg_config命令所在目录在环境变量 ***PATH*** 中，若系统中没有此命令，请安装对应PostgreSQL版本中的 *-dev* 或 *-devel* 软件包。
执行命令 ``` make USE_PGXS=1 ``` 编译版本；执行命令 ``` make USE_PGXS=1 install ``` 安装，若非 *root* 用户安装，报权限不足时，请使用 *sudo*。

示例：在Fedora或RHEL 7系统中，PostgreSQL通过<a href="yum.postgresql.org">yum.postgresql.org</a>软件包安装。
```
sudo dnf install postgresql96-devel
PATH=/usr/pgsql-9.6/bin:$PATH make USE_PGXS=1 clean all
sudo PATH=/usr/pgsql-9.6/bin:$PATH make USE_PGXS=1 install
```

#重要提示

所有发布端和订阅端均需要安装pglogical，例如希望从PostgreSQL9.4同步数据至9.5,必须在9.4版本服务器端安装postgresql94-pglogical，9.5版本服务器端安装postgresql95-pglogical，而不是仅仅安装pglogical-output插件。
更多信息，请参阅/usr/share/doc/pglogical目录下的README.md文件
