# vbdul — VastBase/openGauss/PostgreSQL 数据应急恢复工具

vbdul 是一款数据库应急数据恢复工具（类似 Oracle ODU），支持 VastBase、openGauss 内核版本与 PostgreSQL 11~17。在数据库无法启动、误删表/数据、误 UPDATE/DELETE、无备份的极端场景下，直接从磁盘数据文件、文件系统空闲空间或 WAL 中抢救恢复数据。

> **源代码不公开。** 本仓库仅提供可执行文件发布与使用文档。
> 授权与技术咨询：lxy4@vastdata.com.cn

## 下载

到 [Releases](https://github.com/LiXiangYu0612/vbdul/releases/latest) 下载对应架构的安装包（静态编译 + UPX 压缩，零运行时依赖，解压即用）：

`vbdul-1.1.4-x86_64.tar.gz` / `vbdul-1.1.4-aarch64.tar.gz`

## 授权（License）

正式版按主机指纹授权。首次使用：

```
vbdul> license request          # 生成 license_request.txt（含主机指纹）
vbdul> license show             # 查看授权状态
```

将 `license_request.txt` 发送到 lxy4@vastdata.com.cn 获取 `license.dul`，放到 vbdul 同目录即可。

## 适用场景

| 场景 | 手段 |
|------|------|
| 数据库无法启动、无备份 | 完全离线解析磁盘数据文件（系统表损坏、实例崩溃均可） |
| DROP TABLE / TRUNCATE、无备份 | 文件系统空闲页面碎片扫描，定位被释放数据页还原数据 |
| rm -rf 数据目录 | 目录树扫描 + 按路径提取数据文件 |
| 误 UPDATE / DELETE（有 WAL） | logminer 挖掘 WAL，按事务反向恢复 |
| 有 ProBackup 备份 | 单表选择性恢复，可指定停止 LSN/XID |
| 磁盘坏道、数据库坏块 | 损坏页面/I/O 错误自动跳过，抢救剩余完好数据 |

## 支持矩阵

| 维度 | 支持 |
|------|------|
| 数据库 | VastBase v0/v1/v2/v3/v5/v6（PG11/PG14/PG16/openGauss 6.0/PG17/G100 2.2.5 内核）、PostgreSQL 11~17 |
| 文件系统 | ext4（inode/extent/空闲块）、XFS（AGFL/BNO B+tree/RMAP）、LVM（VG/PV/LV 聚合） |
| 平台 | Linux x86_64、Linux aarch64 |
| 存储引擎 | Heap（行存）；ustore/cstore 暂不支持 |

支持的数据类型：int2/4/8、float4/8、numeric、number、char/varchar/text/nvarchar2、bytea/blob、date/time/timestamp(tz)、bool、jsonb、uuid、money、oid、xml、数组（int[]/text[]/bool[]/numeric[]）、TOAST 大字段（PGLZ 压缩/外部存储，1MB+ 实测）。

## 快速开始

```bash
tar xzf vbdul-1.1.4-x86_64.tar.gz && cd vbdul-1.1.4-x86_64
```

编辑 `config.dul` 指定数据目录与数据库：

```
db_type     vastbase        # 或 postgresql
db_version  v3              # vastbase: v0/v1/v2/v3/v5/v6
data_dir    /home/vastbase/data/vastbase
database    dultest         # 留空 = 多库模式（v1.1.0，见下节）
wal_dir     /home/vastbase/data/vastbase/pg_xlog   # logminer 需要
backup_dir  /backup                                  # ProBackup 恢复需要
```

```
vbdul> unload dict                    # 构建字典（离线解析系统表）
vbdul> list table '%orders%'          # 模糊找表
vbdul> desc public.orders             # 查看表结构
vbdul> unload table public.orders     # 导出数据（COPY 格式，可直接 psql 导入）
```

## 多库支持（v1.1.0）

`database` 留空即进入多库模式：`unload dict` 一次构建全部库的字典，命令用 `db <name>` 子句选库；导出文件带库名前缀（`mydb_public_orders.txt`）互不覆盖。

```
vbdul> unload dict                          # 一次构建全部库的字典
vbdul> list db                              # 列出实例内所有数据库
vbdul> list schema db mydb                  # 列出库内 schema
vbdul> desc public.orders db mydb           # 查看表结构
vbdul> unload table public.orders db mydb   # 导出单表
vbdul> unload db mydb                       # 批量导出整库（expdp 风格输出）
vbdul> unload db a,b,c [meta|data]          # 显式列表；meta 只导 DDL，data 只导数据
vbdul> unload db all                        # 全部库（db_all_exclude 可配排除名单）
vbdul> unload db all except olddb,tmpdb     # all 之上再排除（与 db_all_exclude 并集）
vbdul> imp table public.orders db mydb      # 生成导入脚本
```

`db <name>` 同样适用于 `unload schema` / `verify table` / `dump` / `imp` / `restore` / `recover table` / `logminer`。`database` 有值则保持 v1.0.1 单库行为，命令不带 `db` 子句。

## DROP TABLE / TRUNCATE 恢复（无备份）

被释放的数据页面在文件系统层面仍保留原始内容（直到被覆写）。**恢复动作越早越好，期间避免大量写入。**

```
vbdul> unload dict                              # 生成字典 + filesystem.dul
vbdul> scan filesystem free parallel 4          # 并行扫描空闲空间中的数据页面
vbdul> list free page ext col 5 tlen 100        # 按字段数/tuple长度过滤定位目标表
vbdul> extract free page using vb_ddl           # DDL 匹配提取 + 自动关联 TOAST
vbdul> unload table public.orders recover       # 导出为可导入 SQL
```

## PostgreSQL DROP/TRUNCATE 恢复

`db_type postgresql` 时，`scan filesystem free` 会额外生成 PG 碎片记录（`dict/pg_drop_fragments.dul`），配合 DDL 匹配恢复：

```
vbdul> scan filesystem free parallel 4       # 扫描（生成 pg_drop_fragments.dul）
vbdul> list fragment                         # 列出碎片（F001/F002...）
vbdul> show fragment F002                    # 查看碎片及页面详情
vbdul> match ddl /path/to/ddl.sql            # 用 DDL 匹配碎片 → table_ddl.dul
vbdul> extract pg fragment F002 using ddl /path/to/ddl.sql [--toast F005]   # 提取碎片（含 TOAST）
```

也可以按字典或 DDL 提取整表：`extract pg table <schema.table> using dict` / `using ddl [path]`。

## 数据库无法启动 / 文件被删

```
vbdul> unload dict
vbdul> scan filesystem inode                    # inode 扫描定位已删除文件
vbdul> list filesystem inode
vbdul> extract inode 12345 using vb_ddl         # 提取数据文件 + TOAST
vbdul> unload table public.my_table recover

# rm -rf 数据目录：
vbdul> scan fs /home/vastbase/data              # 按目录树扫描
vbdul> extract fs datadir                       # 按路径提取到 restore/
```

## WAL 误操作回滚（logminer）

前置：config.dul 配置 `wal_dir`，并已执行 `unload dict`。参数为 WAL 段文件名（24 位十六进制，起止段须同 timeline）：

```
vbdul> logminer 00000001000000000000000A 00000001000000000000000C \
       table public.orders start_time '2026-06-20 10:00:00'
vbdul> logminer list table public.orders delete         # 查看被误删的行
vbdul> logminer xid 12345 recover delete                # 按事务恢复：DELETE→数据文件, UPDATE→反向SQL
```

过滤选项：`table <s.t>`（指定表）、`xid <N>`（指定事务，仅 PG）、`start_lsn/stop_lsn <X/XX>`（LSN 区间）、`start_time/stop_time <时间>`（提交时间）、`nobuf`（跳过事务缓冲直接输出 DML）。

## ProBackup 备份恢复

前置：config.dul 配置 `backup_dir`。单表恢复四步（顺序不可颠倒）：

```
vbdul> list backup                              # 列出备份集（list archivelog 列 WAL 归档区间）
vbdul> unload dict backup BACKUP_ID             # 1. 从备份构建字典
vbdul> restore table public.orders              # 2. 还原表数据文件到 restore/
vbdul> recover table public.orders until xid 12345   # 3. WAL 前滚（until lsn X/Y | until time '...' | until xid N）
vbdul> unload table public.orders from backup   # 4. 从 restore/ 导出数据
```

## 数据校验与导入

```
vbdul> verify table public.orders     # 离线校验表数据一致性
vbdul> imp all                        # 为所有已导出表生成 psql 导入脚本
vbdul> imp schema dultest             # 按 schema 生成
vbdul> imp table public.orders        # 按表生成
```

完整命令说明见包内 `help`。

## 安全性

- **严格只读**：所有源数据（设备/数据文件/备份/WAL）仅以只读方式访问，绝不修改原始数据
- 输出仅写入独立目录（`data/`、`dict/`、`extract_file/`、`restore/`）
- 授权基于 Ed25519 签名 + 主机指纹，license 不可伪造、不可跨机使用

## 已知限制

- 仅支持 Heap（行存）引擎，ustore（undo 引擎）与 cstore（列存）暂不支持
- DROP/TRUNCATE 恢复依赖空闲页面未被覆写，恢复窗口内大量写入会降低成功率
