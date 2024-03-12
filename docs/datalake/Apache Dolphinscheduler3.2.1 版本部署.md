## 简介
最近在数据公司的整个数据的中台，梳理整个的数据流和一些公共的指标计算过程，实在是一个比较痛苦的过程。项目的现状，是基于Quatz调度框架，通过Spring反射调研方法执行job，在将计算的结果等异构数据通过datax同步到线下clickhouse存储平台，提供数据分析和报表展示。
当job数量达到上百个，基本上已经很难维护了。

调研了几个分布式调度的工具xxljob、airflow和DolphinScheduler综合比较Apache DolphinScheduler不管是从技术栈还是社区的活跃程度都是比较符合的。
![schedule-sys-compare.png](img%2Flenovo-dolphin%2Fschedule-sys-compare.png)

Apache DolphinScheduler 是一个分布式易扩展的可视化DAG工作流任务调度开源系统。适用于企业级场景，提供了一个可视化操作任务、工作流和全生命周期数据处理过程的解决方案。

Apache DolphinScheduler 旨在解决复杂的大数据任务依赖关系，并为应用程序提供数据和各种 OPS 编排中的关系。 解决数据研发ETL依赖错综复杂，无法监控任务健康状态的问题。 DolphinScheduler 以 DAG（Directed Acyclic Graph，DAG）流式方式组装任务，可以及时监控任务的执行状态，支持重试、指定节点恢复失败、暂停、恢复、终止任务等操作。

官方文档：
https://dolphinscheduler.apache.org/zh-cn



## Apache DolphinScheduler3.2.1最新版本安装部署

### 1、单机Docker部署

**前置条件**

- 需要安装 Docker 1.13.1 以上版本，以及 Docker Compose 1.28.0 以上版本。

```bash
$ DOLPHINSCHEDULER_VERSION=3.2.1
$ docker run --name dolphinscheduler-standalone-server -p 12345:12345 -p 25333:25333 -d apache/dolphinscheduler-standalone-server:"${DOLPHINSCHEDULER_VERSION}"
```
浏览器访问地址 http://localhost:12345/dolphinscheduler/ui 即可登录系统 UI。默认的用户名和密码是 admin/dolphinscheduler123

![dolphinscheduler-login.png](img%2Fdolphinscheduler-bushu%2Fdolphinscheduler-login.png)


### 2、伪集群部署Pseudo-Cluster

 **前置准备工作**
伪分布式部署 DolphinScheduler 需要有外部软件的支持

- JDK：下载JDK (1.8+)，安装并配置 JAVA_HOME 环境变量，并将其下的 bin 目录追加到 PATH 环境变量中。如果你的环境中已存在，可以跳过这步。
- 二进制包：在下载页面下载 DolphinScheduler 选择二进制包 https://dolphinscheduler.apache.org/zh-cn/download/3.2.1
- 数据库：PostgreSQL (8.2.15+) 或者 MySQL (5.7+)，两者任选其一即可，如 MySQL 则需要 JDBC Driver 8.0.16， 我的环境是MySQL
- 注册中心：ZooKeeper (3.8.0+)，下载地址 https://zookeeper.apache.org/releases.html
- 进程树分析 Centos8 安装 psmisc
注意: DolphinScheduler 本身不依赖 Hadoop、Hive、Spark，但如果你运行的任务需要依赖他们，就需要有对应的环境支持

**如何切换数据源MySQL?**

官方也有很详细的介绍， 

https://github.com/apache/dolphinscheduler/blob/3.2.1-release/docs/docs/zh/guide/howto/datasource-setting.md

1. 下载MySQL8驱动
   如果使用 MySQL 需要手动下载 [mysql-connector-java](https://downloads.mysql.com/archives/c-j/) 驱动 (8.0.16) 并移动到 DolphinScheduler 的每个模块的 libs 目录下，其中包括 api-server/libs 和 alert-server/libs 和 master-server/libs 和 worker-server/libs 和 tools/libs。

2. 创建数据库和用户

```bash
mysql -uroot -p

mysql> CREATE DATABASE dolphinscheduler DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

# 修改 {user} 和 {password} 为你希望的用户名和密码
mysql> CREATE USER '{user}'@'%' IDENTIFIED BY '{password}';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO '{user}'@'%';
mysql> CREATE USER '{user}'@'localhost' IDENTIFIED BY '{password}';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO '{user}'@'localhost';
mysql> FLUSH PRIVILEGES;
```

3. 修改环境变量配置

```bash
mysql -uroot -p

mysql> CREATE DATABASE dolphinscheduler DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

# 修改 {user} 和 {password} 为你希望的用户名和密码
mysql> CREATE USER '{user}'@'%' IDENTIFIED BY '{password}';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO '{user}'@'%';
mysql> CREATE USER '{user}'@'localhost' IDENTIFIED BY '{password}';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO '{user}'@'localhost';
mysql> FLUSH PRIVILEGES;

```

4. 初始化数据库脚本
```bash
bash tools/bin/upgrade-schema.sh
```

**准备 DolphinScheduler 启动环境**
1. 配置用户免密及权限
创建部署用户，并且一定要配置 sudo 免密。以创建 dolphinscheduler 用户为例
```bash
# 创建用户需使用 root 登录
useradd dolphinscheduler

# 添加密码
echo "dolphinscheduler" | passwd --stdin dolphinscheduler

# 配置 sudo 免密
sed -i '$adolphinscheduler  ALL=(ALL)  NOPASSWD: NOPASSWD: ALL' /etc/sudoers
sed -i 's/Defaults    requirett/#Defaults    requirett/g' /etc/sudoers

# 修改目录权限，使得部署用户对二进制包解压后的 apache-dolphinscheduler-*-bin 目录有操作权限
chown -R dolphinscheduler:dolphinscheduler apache-dolphinscheduler-*-bin
chmod -R 755 apache-dolphinscheduler-*-bin
```

2. 配置机器 SSH 免密登陆

由于安装的时候需要向不同机器发送资源，所以要求各台机器间能实现 SSH 免密登陆。配置免密登陆的步骤如下

```bash
su dolphinscheduler

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**_注意_:** 配置完成后，可以通过运行命令 ssh localhost 判断是否成功，如果不需要输入密码就能 ssh 登陆则证明成功

3. 启动 zookeeper
进入 zookeeper 的安装目录，将 zoo_sample.cfg 配置文件复制到 conf/zoo.cfg，并将 conf/zoo.cfg 中 dataDir 中的值改成 dataDir=./tmp/zookeeper

```bash
# 启动 zookeeper
./bin/zkServer.sh start
```

4. 修改相关配置
完成基础环境的准备后，需要根据你的机器环境修改配置文件。配置文件可以在目录 bin/env 中找到，他们分别是 并命名为 install_env.sh 和 dolphinscheduler_env.sh。

4.1 修改 install_env.sh 文件
文件 install_env.sh 描述了哪些机器将被安装 DolphinScheduler 以及每台机器对应安装哪些服务。您可以在路径 bin/env/install_env.sh 中找到此文件，可通过以下方式更改 env 变量,export <ENV_NAME>=，配置详情如下。
![install_env.png](img%2Fdolphinscheduler-bushu%2Finstall_env.png)

4.2 修改 dolphinscheduler_env.sh 文件
文件 ./bin/env/dolphinscheduler_env.sh 描述了下列配置：

DolphinScheduler 的数据库配置，详细配置方法见[初始化数据库]
一些任务类型外部依赖路径或库文件，如 JAVA_HOME 和 SPARK_HOME都是在这里定义的
如果您不使用某些任务类型，您可以忽略任务外部依赖项，但您必须根据您的环境更改 JAVA_HOME、注册中心和数据库相关配置。
![dolphinscheduler_env.png](img%2Fdolphinscheduler-bushu%2Fdolphinscheduler_env.png)

5. 启动 DolphinScheduler
使用上面创建的部署用户运行以下命令完成部署，部署后的运行日志将存放在 logs 文件夹内
```bash
bash ./bin/install.sh
```
6. 登录 DolphinScheduler
浏览器访问地址 http://localhost:12345/dolphinscheduler/ui 即可登录系统 UI。默认的用户名和密码是 admin/dolphinscheduler123

7. 启停服务
```bash
# 一键停止集群所有服务
bash ./bin/stop-all.sh

# 一键开启集群所有服务
bash ./bin/start-all.sh

# 启停 Master
bash ./bin/dolphinscheduler-daemon.sh stop master-server
bash ./bin/dolphinscheduler-daemon.sh start master-server

# 启停 Worker
bash ./bin/dolphinscheduler-daemon.sh start worker-server
bash ./bin/dolphinscheduler-daemon.sh stop worker-server

# 启停 Api
bash ./bin/dolphinscheduler-daemon.sh start api-server
bash ./bin/dolphinscheduler-daemon.sh stop api-server

# 启停 Alert
bash ./bin/dolphinscheduler-daemon.sh start alert-server
bash ./bin/dolphinscheduler-daemon.sh stop alert-server
```

## 3.FAQ 部署遇到的问题
1. 如何开启develop mode？
配置一个简单的从预发布PRE环境MySQL 通过datax同步到线下clickhouse，
解决方法参考：
https://github.com/apache/dolphinscheduler/discussions/15293
```bash
# 修改配置参数在 worker-server 的conf common.property文件中修改 development.state=true
# development state
development.state=true

```
如图打开开发模式可以直接到目录打开json，通过脚本执行
![open_develop_mode.png](img%2Fdolphinscheduler-bushu%2Fopen_develop_mode.png)

2. ${PYTHON_LAUNCHER} ${DATAX_LAUNCHER} 行4: --jvm=-Xms1G -Xmx1G: 未找到命令
打开开发这模式看到对应执行的命令，直接手工执行也是报错
```text
[INFO] 2024-03-12 22:13:07.999 +0800 - ****************************** Script Content *****************************************************************
[INFO] 2024-03-12 22:13:08.000 +0800 - Executing shell command :sudo -u default /tmp/dolphinscheduler/exec/process/default/12902998429184/12903370942208_2/7/7/7_7.sh
[INFO] 2024-03-12 22:13:08.004 +0800 - process start, process id is: 248648
[INFO] 2024-03-12 22:13:09.018 +0800 -  -> 
	/tmp/dolphinscheduler/exec/process/default/12902998429184/12903370942208_2/7/7/7_7.sh:行4: --jvm=-Xms1G -Xmx1G: 未找到命令
[INFO] 2024-03-12 22:13:09.023 +0800 - process has exited. execute path:/tmp/dolphinscheduler/exec/process/default/12902998429184/12903370942208_2/7/7, processId:248648 ,exitStatusCode:127 ,processWaitForStatus:true ,processExitValue:127
[INFO] 2024-03-12 22:13:09.024 +0800 - ***********************************************************************************************
```
解决方法参考：
https://github.com/apache/dolphinscheduler/issues/15065

解决方案在dolphinscheduler_env.sh配置的环境变量没有生效，执行的脚本命令 sudo -u default 所以最好在/etc/profile里面配置然后source /etc/profile对全部用户生效。
检查配置：echo ${PYTHON_LAUNCHER}
![datax-env.png](img%2Fdolphinscheduler-bushu%2Fdatax-env.png)

![etc_profile.png](img%2Fdolphinscheduler-bushu%2Fetc_profile.png)

## 总结
通过Apache DolphinScheduler分布式调度，完成数据集成，可视化调度，任务的监控运维统计，给中小企业构建大数据中台降低了整体的搭建开发的难度和开发的成本。

## 参考文档
[联想基于Apache DolphinScheduler构建统一调度中心的应用实践](https://zhuanlan.zhihu.com/p/611680333) \
[Apache DolphinScheduler 官方文档](https://dolphinscheduler.apache.org/zh-cn/docs/3.2.1)