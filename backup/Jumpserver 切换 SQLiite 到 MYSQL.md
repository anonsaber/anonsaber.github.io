## 背景

因为升级和分布式部署，现在需要将 jumpserver 的数据库从 SQLite 切换为 MySQL/Mariadb

网上搜了一圈，没见到有人写这个，可能没有倒霉蛋需要填这个坑吧。

不知道是哪一任从若干年前 1.4.x 版本发布的时候使用了 SQLite 做数据库一直不更新的用到现在，有几百台设备，几十个设备分组，几十个权限分组。

现在的情况是：如果不能很好的把数据库从 SQLite 迁移到 MYSQL/MariaDB，并且升级到一个比较新的版本的话，就得新搭建一个新版本然后照着之前的一个一个手工录入。

X，想想也是恶心的事儿，何况有些私钥可能没人有保留。

折腾了半天，验证了可行的方案，提供给有需要的倒霉蛋。

以下步骤在某组织架构不怎么靠谱的盈利的勉强算得上是互联网中厂的生产环境做过验证(v1.4.10-2 > v2.28.7)。

## 不完全停掉旧版本

具体来说，就是保留 redis 和 mysql，停掉其他的。

如果使用源码方式部署找到 core 服务所在的目录，执行下面的命令

```
python manage.py dumpdata --natural-foreign --exclude admin.logentry --exclude sessions.session --exclude contenttypes --exclude auth.permission --indent 4 > /tmp/dump.json
```

如果使用 docker 部署，想想办法把 core 服务容器启动命令换成 sleep 12288 （总之不要让服务启动），然后也是执行上面的命令。至于导出的 dump.json 要怎么从容器里面弄出来，这里不做解释。
创建数据库

这里创建了 MariaDB 5.5 版本，或许可以直接使用 10.3 以后的版本，但是我没有测试过（因为我最开始没有看 jumpserver 2.8 版本需要 MariaDB 大于等于 10.3 ，于是后面有个数据库升级的操作，或许可以直接用 10.3，或许）

```sql
create database jumpserver default charset 'utf8';
create user 'jumpserver'@'%' identified by 'jumpserver';
grant all on jumpserver.* to 'jumpserver'@'%';
flush privileges;
```

## 修改当前旧版本配置文件

注释掉 SQLite，调整为 MySQL

```
# SQLite setting:
# 使用单文件sqlite数据库
# DB_ENGINE: sqlite3
# DB_NAME: /opt/jumpserver/jumpserver.db

# MySQL or postgres setting like:
# 使用Mysql作为数据库
DB_ENGINE: mysql
DB_HOST: 127.0.0.1
DB_PORT: 3306
DB_USER: jumpserver
DB_PASSWORD: jumpserver
DB_NAME: jumpserver
```

## 初始化新数据库

这里不能直接启动 jumpserver，使用源码部署的话用下面的命令。

```bash
python apps/manage.py migrate
python apps/manage.py flush
python apps/manage.py loaddata /tmp/dump.json
```

如果使用 docker 部署，想想办法把 core 服务容器启动命令换成 sleep 12288（总之不要让服务启动），然后也是执行上面的命令。至于怎么把 dump.json 弄到容器里面，这里也不做解释。
启动当前旧版本

现在，将所有旧版本的 jumpserver 服务启动，数据库就完成了切换。

## 备份数据库

测试通过后，停掉除了 MYSQL/MariaDB 的所有服务（你也可以使用 xtrabackup 之类的其他东西做热备份），然后执行数据库备份。

```bash
mysqldump --databases jumpserver -h127.0.0.1 -uroot -p > /tmp/database-backup.sql
```

## 修改数据库字符集

来自官方的迁移文档：

```bash
# 如果你不需要或不想处理数据库字符集可以跳过此步骤, 保证迁移前后的数据库字符集一样即可.
if grep -q 'COLLATE=utf8_bin' /opt/jumpserver.sql; then
    cp /opt/jumpserver.sql /opt/jumpserver_bak.sql
    sed -i 's@ COLLATE=utf8_bin@@g' /opt/jumpserver.sql
    sed -i 's@ COLLATE utf8_bin@@g' /opt/jumpserver.sql
else
    echo "备份数据库字符集正确";
fi
```

## 升级 MariaDB 版本到 10.3

```sql
drop database jumpserver;
create database jumpserver default charset 'utf8';
GRANT ALL PRIVILEGES ON *.* TO 'jumpserver'@'%' IDENTIFIED BY 'jumpserver' WITH GRANT OPTION;
use jumpserver
source /tmp/database-backup.sql
flush privileges;
```

这里用了将备份还原的方式，你也可以用其他方式升级上来。

## 更新新版本服务

这部分请参考官方文档，该注意的地方都要注意，比如 SECRET_KEY 之类的。

本文同步发表于

https://bbs.fit2cloud.com/t/topic/639/1

<!-- ##{"timestamp":1704643200}## -->