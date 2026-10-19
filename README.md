# vbdul - VastBase/openGauss 数据应急恢复工具

vbdul 是一款 VastBase/openGauss 数据库应急数据恢复工具，类似 Oracle ODU。在数据库无法启动、误删表/数据、无备份的极端场景下，直接从磁盘数据文件中抢救恢复数据。

## 适用场景

- **数据库无法启动、无备份**：系统表损坏、实例崩溃时完全离线操作，直接解析磁盘数据文件
- **DROP TABLE / TRUNCATE、无备份**：文件系统级空闲页面扫描，定位被释放的数据页面还原数据；有 ProBackup 备份时按时间点回退
- **磁盘坏道、数据库坏块、无备份**：损坏页面自动跳过，尽可能从剩余完好页面抢救数据

# 使用手册

适用版本：v1.1.14 及以上

## 1. 部署

```bash
tar xzf vbdul-1.1.14-x86_64.tar.gz        # aarch64 服务器用 vbdul-1.1.14-aarch64.tar.gz
cd vbdul-1.1.14-x86_64
vi config.dul                             # 按第 2 节配置
./vbdul
```

正式版首次运行处于受限模式（只能 license / config / help）。执行 `license request`
生成 license_request.txt，发给供应商换取 license.dul，放到 vbdul 同目录后重新运行
即可。Beta 版无需 license。

## 2. 配置 config.dul（vi 编辑）

用 vi 打开解压目录下的 config.dul，按"配置项 值"逐行填写。关键配置：

| 配置项 | 必填 | 说明 |
|--------|------|------|
| db_type | 是 | `vastbase` 或 `postgresql`，按实例内核填；填错直接报错退出 |
| db_version | 是 | 内核版本代号，见下表 |
| data_dir | 是 | 数据目录（VastBase 如 `/home/vastbase/data/vastbase`；PG 如 `/var/lib/pgsql/17/data`） |
| database | 否 | 目标库名。默认不配置（操作整个实例）；配置后只操作该库且命令省 db 子句，见 2.1 |
| output_dir | 否 | 导出目录，默认 `data` |
| wal_dir | 用 logminer 必填 | WAL 目录（VastBase 是 `pg_xlog`，PG 是 `pg_wal`），也可指向 WAL 归档目录 |
| backup_dir | 用备份恢复必填 | ProBackup 备份目录 |
| backup_wal_dir | 用备份恢复必填 | 备份对应的 WAL 归档目录 |
| delimiter | 否 | 导出字段分隔符，默认 `\|` |
| charset | 否 | 字符集，默认 `UTF8` |
| ora_number | 否 | VastBase Oracle 兼容 NUMBER 的显示方式，默认 `on` |
| db_all_exclude | 否 | 建字典时默认跳过的库，默认 `template0,template1,template2` |
| block_size / byte_order | 否 | 默认 `8192` / `little`，一般不改 |

db_version 取值：

| db_type | db_version | 对应版本 |
|---------|------------|----------|
| vastbase | v0 | E100 V3（AtlasDB 3.0.3.8，PG11 内核） |
| vastbase | v1 | E100 V4（PG14 内核） |
| vastbase | v2 | E100 V5（PG16 内核） |
| vastbase | v3 | G100 2215/308/309（openGauss 内核） |
| vastbase | v5 | G100 V5（PG17 内核） |
| vastbase | v6 | G100 2.2.5（openGauss 内核） |
| postgresql | pg10 ~ pg17 | 对应 PostgreSQL 10 ~ 17 |

### 2.1 默认（不配置 database）与配置了的区别

**默认不配置 database——操作整个实例，对象级命令用 db 子句指明库：**

```
vbdul> unload dict                        # 给实例内所有库建字典（template0/1/2 除外）
vbdul> list db                            # 列出实例内所有库
vbdul> desc public.orders db dultest      # 查表结构
vbdul> unload table public.orders db dultest
vbdul> unload db dultest                  # 整库导出；unload db all 导出全部库
vbdul> imp table public.orders db dultest
```

vastbase / postgres 等实例自带库默认也会建字典，`unload table xxx db vastbase`
可直接使用。导出文件名带库名前缀，例如 `dultest_public_orders.txt`、
`dultest_public_orders.sql`。db 子句位置不拘（写在前后均能正确解析），
本手册统一紧跟对象写：`unload table public.orders db dultest recover`。

**配置了 database（例如 `database dultest`）——只操作这一个库，命令里不用再写
db 子句：**

```
vbdul> unload dict                        # 只给 dultest 建字典
vbdul> unload table public.orders         # 等价于 ... db dultest
vbdul> imp                                # 只见 dultest 的已导出表
```

## 3. 基本流程（表还在，导出数据）

以 dultest 库 public.orders 为例（配置了 database 时去掉 db 子句）：

```
vbdul> unload dict                        # 1. 构建字典，首次必做
vbdul> desc public.orders db dultest      # 2. 查看表结构
vbdul> unload table public.orders db dultest   # 3. 导出 → data/dultest_public_orders.txt/.sql
vbdul> imp table public.orders db dultest # 4. 生成该表的导入脚本
```

### 3.1 imp：生成导入脚本

imp 为 **unload 导出的表**生成 psql 导入脚本（.sh + .sql 成对落在 data/），
按范围三种写法：

| 命令 | 范围 | 产物（data/ 下） |
|------|------|------------------|
| `imp` | 全部已导出的表 | `import_all.sh` + `import_all.sql` |
| `imp schema app` | 该 schema 的已导出表 | `import_app.sh` + `import_app.sql` |
| `imp table public.orders` | 单张表 | `import_public_orders.sh` + `import_public_orders.sql` |

多库形态下 imp 同样带 db 子句，如 `imp table public.orders db dultest`。
没有导出记录的表不会出现在任何 imp 脚本里。

导入（任选其一）：

```bash
cd data
./import_all.sh <库名> <主机> <端口> <用户>
# 或者
psql -h <主机> -p <端口> -U <用户> -d <库名> -f import_all.sql
```

`imp` 只处理 unload 导出的表；recover / logminer 的输出不会进导入脚本（见第 4、6 节）。

## 4. DROP TABLE / TRUNCATE 恢复（无备份）

DROP / TRUNCATE 之后，数据库释放的数据页在文件系统层面仍保留原内容，直到被新数据
覆写。vbdul 扫描这些空闲页面，从中还原表数据。

恢复前提：

- 误操作后**没有大量写入**，页面尚未被覆写；越早恢复成功率越高（建议立即停止或
  减少该实例的写入）
- 大致记得表名和字段数（用于过滤定位）

步骤（以 public.orders，5 个字段为例）：

```
vbdul> unload dict                        # 1. 建字典（同时生成 filesystem.dul 设备清单）
vbdul> scan filesystem                    # 2. 扫描空闲页（页扫+inode+目录树+WAL 挖矿，
                                          #    空闲页结果用于本恢复）
vbdul> list free page ext col 5           # 3. 可选：按字段数过滤定位（可再加 tlen <行长>）
vbdul> extract free page using vb_ddl     # 4. 提取并按 DDL 认领 → extract_file/
                                          #    PostgreSQL 实例用 pg_ddl
vbdul> desc public.orders db dultest recover   # 5. 核对认领出的表结构
vbdul> unload table public.orders db dultest recover   # 6. 导出 → data/dultest_recover_public_orders.txt
```

第 4 步也可以只提取指定碎片：`extract free page ext 1,2,3 using vb_ddl`。

导出的 .txt 是 COPY 文本格式，直接导入（imp 不会导入 recover 产物）：

```bash
cd data
psql -d <库名> -c "\copy recover_public_orders FROM 'dultest_recover_public_orders.txt' WITH (FORMAT text, DELIMITER '|')"
```

导入前先建好表结构（可用导出的同名 .sql）。

## 5. TRUNCATE 恢复（有 ProBackup 备份）

有备份时用 WAL 回放把表恢复到误操作之前的状态，结果最可靠。这组命令目前要求
config.dul **配置了 database**，并配置备份目录：

```
database        dultest
backup_dir      /home/vastbase/backup
backup_wal_dir  /home/vastbase/backup/wal
restore_dir     restore
```

步骤（顺序不可颠倒）：

```
vbdul> list backup                                       # 1. 列出可用备份集
vbdul> unload dict backup                                # 2. 从备份构建字典
vbdul> restore table public.orders                       # 3. 还原表数据文件到 restore/
vbdul> recover table public.orders until time '2026-10-15 10:00:00'
                                                          # 4. WAL 回放到指定时间点；
                                                          #    也可 until xid <事务号> 或 until lsn <X/Y>
vbdul> unload table public.orders from backup            # 5. 导出恢复结果
```

## 6. Logminer（WAL DML 挖掘）

从 WAL 挖掘 INSERT / DELETE / UPDATE，定位误操作事务，恢复被误删或误改的行。
前置：config.dul 配置 `wal_dir`，并已执行 `unload dict`。

```
wal_dir    /home/vastbase/data/vastbase/pg_xlog    # PostgreSQL 为 .../pg_wal；也可指向归档目录
```

挖掘（起止参数可以是 **WAL 段名**或 **LSN**，二者自动识别；默认形态须带
`db <库名>`，不带会直接报错，不会静默挖错库）：

```
vbdul> logminer 000000010000000000000010 000000010000000000000018 db dultest   # 段名（24 位十六进制，ls wal_dir 可见）
vbdul> logminer 000000010000000000000010 000000010000000000000010 db dultest   # 只挖一个段：起止传同一个段名
vbdul> logminer 0/4EDD1E8 0/4EDDF90 db dultest                  # LSN：自动换算所在段，按精确 LSN 区间过滤
vbdul> logminer 000000010000000000000010 000000010000000000000018 table public.orders db dultest start_time '2026-10-15 09:00:00'
```

可选过滤：`table <schema.table>`、`xid <N>`（仅 PG）、`start_lsn` / `stop_lsn`、
`start_time` / `stop_time`、`nobuf`（不缓冲直接输出 DML）。
配置了 database 时以上命令可省略 db 子句。

关于 UPDATE / DELETE：VastBase 引擎的 WAL 不记录旧行镜像，因此 UPDATE 的
REDO 带新值、DELETE 的 REDO 只到表名，两者 UNDO 列标注"不可恢复"；
`logminer xid ... recover` 遇到这类记录会明确报"匹配到但行值不可恢复"。
INSERT 的 REDO / UNDO 完整。

定位与恢复：

```
vbdul> logminer list table public.orders db dultest delete   # 按事务列出该表的 DELETE
vbdul> logminer xid 4832 recover delete db dultest           # 恢复该事务删掉的行 → 数据行 .txt
vbdul> logminer xid 4832 recover update db dultest           # 反向 UPDATE 语句 → .sql
```

logminer 输出在 data/ 下（带库名前缀），imp 不会导入这些文件，人工核对后执行。

## 注意事项

- vbdul 以只读方式访问设备和数据文件，不会修改原始数据；输出只写入自身目录
  （`data/`、`dict/`、`extract_file/`、`restore/`）
- 操作前建议先备份原始数据文件
- 仅支持 Heap（行存）引擎；ustore（undo 引擎）、cstore（列存引擎）暂不支持
- 更多命令在 vbdul 内执行 `help` 查看

## 支持平台

| 平台 | 状态 |
|------|------|
| Linux x86_64 | 已支持 |
| Linux aarch64 | 已支持 |
