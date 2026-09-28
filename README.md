# vbdul - VastBase/openGauss 数据应急恢复工具

vbdul 是一款 VastBase/openGauss 数据库应急数据恢复工具，类似 Oracle ODU。在数据库无法启动、误删表/数据、无备份的极端场景下，直接从磁盘数据文件中抢救恢复数据。

## 适用场景

### 数据库无法启动、无备份

系统表损坏、实例崩溃、无法进入数据库时，vbdul 完全离线操作，直接解析磁盘数据文件。精确解析 Heap 页面、Tuple 字段、MVCC 事务可见性，从原始页面还原数据。

### DROP TABLE / TRUNCATE、无备份

通过文件系统级空闲页面碎片扫描（ext4/XFS），定位被释放的数据页面，从磁盘原始页面还原数据。支持分区表、TOAST 大字段（1MB+）、JSONB、数组等复杂类型。

### 磁盘坏道、数据库坏块、无备份

损坏页面自动跳过，I/O 报错自动跳过，尽可能从剩余完好页面中抢救数据。支持 ext4/XFS 双文件系统深度恢复：B+tree 遍历、AGFL 环形队列扫描、inode 反向映射。支持 LVM：VG/PV/LV 自动识别，多设备聚合扫描。

## 核心价值

- **无备份下的最后一道防线**：数据库不可用且无备份时，唯一能从磁盘原始数据中抢救恢复的手段
- **全功能覆盖**：字典自动构建、Heap 页面解析、MVCC 可见性判断、TOAST 大字段解压、分区表、JSONB、数组类型、ext4/XFS 文件系统级恢复
- **坏块容错**：损坏页面跳过不中断，最大程度挽回可恢复数据

## 关键收益

| 场景 | 成效 |
|------|------|
| 无备份 + 数据库无法启动 | 直接从磁盘抢救数据，最后一道防线 |
| 无备份 + DROP/TRUNCATE | 空闲页面碎片扫描定位被释放数据，最大程度挽回损失 |
| 无备份 + 坏道/坏块 | 损坏页面、I/O 报错自动跳过，抢救所有可恢复数据 |
| 部署门槛 | 打包即用，拷贝到服务器即可操作 |

## 快速开始

### 1. 部署

```bash
# 解压安装包
tar xzf vbdul-Beta_1.0.0-x86_64.tar.gz
cd vbdul-Beta_1.0.0-x86_64
```

### 2. 配置

编辑 `config.dul`，指定 VastBase 数据目录和数据库名：

```
data_dir    /home/vastbase/data/vastbase
database    dultest
```

### 3. 恢复数据

```bash
./vbdul
```

```
vbdul> unload dict                          # 构建字典（自动读取系统表）
vbdul> desc public.my_table                 # 查看表结构
vbdul> unload table public.my_table         # 导出数据到 data/ 目录
```

导出文件为标准 COPY 格式 SQL，可直接 `psql -f` 导入。

## 支持的数据类型

| 类别 | 类型 |
|------|------|
| 整数 | int2, int4, int8 |
| 浮点 | float4, float8 |
| 数值 | numeric, number |
| 字符 | char, varchar, text, nvarchar2 |
| 二进制 | bytea, blob |
| 时间 | date, time, timestamp, timestamptz |
| 布尔 | bool |
| JSON | jsonb |
| 大字段 | TOAST（PGLZ 压缩/外部存储） |
| 数组 | int[], text[], bool[], numeric[] |
| 其他 | uuid, money, oid, xml |

## DROP TABLE / TRUNCATE 恢复流程

DROP TABLE 和 TRUNCATE 后，数据库释放的数据页面在文件系统层面仍保留原始内容（直到被覆写）。vbdul 通过空闲页面碎片扫描，定位这些被释放但尚未覆写的数据页面，从中还原表数据。

### 恢复步骤

#### 1. 构建字典

```
vbdul> unload dict
```

自动从系统表生成字典文件，同时生成 `dict/filesystem.dul` 记录存储设备信息。

#### 2. 扫描空闲页面碎片

```
vbdul> scan filesystem free
```

扫描文件系统空闲空间（ext4 空闲块 / XFS AGFL + BNO B+tree），识别包含 VastBase 数据页面的空闲区域，结果写入 `dict/xfs_free_ext.dul`。

支持并行扫描加速：
```
vbdul> scan filesystem free parallel 4
```

也可以同时扫描 inode 和空闲碎片：
```
vbdul> scan filesystem              # 全量扫描：inode + 空闲碎片
```

#### 3. 过滤和定位目标表

```
vbdul> list free page ext col 5 [tlen 100]
```

按字段数和 tuple 长度过滤 `xfs_free_ext.dul`，缩小目标范围。

#### 4. 提取数据页面

根据过滤结果，提取空闲页面中的数据文件：

```
vbdul> extract free page ext 1,2,3 using vb_ddl
```

按 DDL 解析提取，输出到 `extract_file/` 目录，生成 `{schema}_{table}_heap_ext{N}_{start}_{end}.dat` 文件。

也可以全部提取：
```
vbdul> extract free page using vb_ddl
```

#### 5. 导出恢复数据

```
vbdul> unload table public.orders recover
```

自动从 `extract_file/` 目录发现已提取的数据文件，结合 DDL 信息，解析 tuple 并导出为可导入的 SQL 文件。

查看恢复的表结构：
```
vbdul> desc public.orders recover
```

### 恢复前提

- DROP/TRUNCATE 后**未大量写入新数据**（否则空闲页面可能已被覆写）
- 知道被删除表的 DDL（表名、字段定义）
- 恢复越早操作越好，减少数据被覆写的风险

### 完整示例

```
vbdul> unload dict
vbdul> scan filesystem free
vbdul> list free page ext col 5
vbdul> extract free page ext 1,2,3 using vb_ddl
vbdul> unload table public.orders recover
```

导出文件位于 `data/recover_public_orders.txt`，可直接导入数据库。

## Inode 恢复流程（数据库无法启动）

当数据库无法启动但数据文件尚未被删除时，可通过 inode 扫描定位数据文件：

```
vbdul> unload dict
vbdul> scan filesystem inode
vbdul> list filesystem inode
vbdul> extract inode 12345
vbdul> unload table public.my_table recover
```

## 文件系统恢复

vbdul 支持直接访问块设备，在文件被删除或文件系统损坏时恢复数据。

支持的文件系统：
- **ext4**：inode 扫描、extent tree 遍历、空闲块扫描、已删除文件检测
- **XFS**：AGFL 环形队列扫描、BNO/CNT B+tree 遍历、RMAP 反向映射、inode 扫描、空闲碎片扫描
- **LVM**：VG/PV/LV 自动识别，多设备聚合扫描

## REPL 命令

### 基础命令

| 命令 | 说明 |
|------|------|
| `help` | 显示帮助 |
| `config load/show/save` | 配置管理 |
| `unload dict` | 构建字典（读取系统表，生成 .dul 文件） |
| `desc <schema.table>` | 查看表结构（含分区信息） |
| `unload table <schema.table>` | 导出表数据 |
| `unload table <oid> column <types>` | 无字典模式导出（指定 OID 和类型） |
| `dump filenodeid <id> block <n>` | 查看页面结构 |
| `exit / quit` | 退出 |

### 文件系统扫描与恢复

| 命令 | 说明 |
|------|------|
| `scan filesystem` | 全量扫描：inode + 空闲碎片 |
| `scan filesystem inode` | inode 扫描：定位已删除文件 |
| `scan filesystem inode <num>` | 直接解析指定 inode 号 |
| `scan filesystem free [parallel N]` | 空闲碎片扫描：查找 VB 数据页面 |
| `list filesystem inode` | 列出 inode 扫描结果 |
| `list free page ext col <n> [tlen <m>]` | 按字段数和 tuple 长度过滤空闲碎片 |
| `extract inode <num> [using vb_ddl]` | 提取 inode 数据文件 |
| `extract free page ext <n> [using vb_ddl]` | 提取指定空闲碎片的数据 |
| `extract free page using vb_ddl` | 全部空闲碎片提取 |
| `unload table <name> recover` | 从 extract_file/ 导出恢复数据 |
| `desc <name> recover` | 查看恢复表的结构 |

### 数据校验与导入脚本

| 命令 | 说明 |
|------|------|
| `verify table <schema.table>` | 离线校验表数据一致性 |
| `imp [all]` | 为所有已导出表生成导入脚本 |
| `imp schema <name>` | 为指定 schema 的表生成导入脚本 |
| `imp table <schema.table>` | 为指定表生成导入脚本 |

`imp` 扫描 `data/` 目录下的导出文件，结合 config.dul 中的 `database` 生成 psql 导入脚本。

### ProBackup 备份恢复

从 probackup 备份集选择性恢复单表，并用 WAL 前滚到指定时间点。config.dul 需配置 `backup_dir`（备份目录）。

| 命令 | 说明 |
|------|------|
| `unload dict backup [BACKUP_ID]` | 从 probackup 备份构建字典（backup_*.dul） |
| `list backup` | 列出 backup_dir 中的备份集 |
| `list archivelog` | 列出 WAL 归档区间 |
| `show backup <id>` | 查看备份集详情 |
| `restore table <schema.table>` | 从备份还原表数据文件到 restore/ |
| `recover table <schema.table> until lsn <X/Y>` | WAL 前滚到指定 LSN |
| `recover table <schema.table> until time 'YYYY-MM-DD HH:MM:SS'` | WAL 前滚到指定时间 |
| `recover table <schema.table> until xid <N>` | WAL 前滚到指定事务 |

单表恢复完整流程（前置顺序不可颠倒）：

```
vbdul> unload dict backup                                              # 1. 从备份构建字典
vbdul> restore table public.orders                                     # 2. 还原表数据文件
vbdul> recover table public.orders until time '2026-08-19 10:00:00'    # 3. WAL 前滚
vbdul> unload table public.orders from backup                          # 4. 从 restore/ 导出数据
```

### Logminer（WAL DML 挖掘）

从 WAL 日志挖掘 DML（INSERT/DELETE/UPDATE），VB/PG 内核自动识别。前置：config.dul 配置 `wal_dir`，并已执行 `unload dict`（按 relnode 反查表名）。

主命令 `logminer <start_wal> <end_wal>`（WAL 段名，如 `00000001000000000000000A`），支持以下过滤选项：

| 选项 | 说明 |
|------|------|
| `table <schema.table>` | 只挖掘指定表 |
| `xid <N>` | 只挖掘指定事务（仅 PG） |
| `start_lsn <X/XX>` / `stop_lsn <X/XX>` | 按 LSN 区间过滤 |
| `start_time <YYYY-MM-DD [HH:MM:SS]>` / `stop_time <...>` | 按提交时间过滤 |
| `nobuf` | 跳过事务缓冲，直接输出 DML |

辅助命令：

| 命令 | 说明 |
|------|------|
| `logminer list table <schema.table> [insert\|delete\|update]` | 按事务列出表上的 DML 操作 |
| `logminer xid <XID> recover [delete\|update]` | 恢复该事务的数据：DELETE 生成数据行 .txt，UPDATE 生成反向 SQL .sql |

误删数据挖掘示例：

```
vbdul> logminer 00000001000000000000000A 00000001000000000000000C table public.orders
vbdul> logminer list table public.orders delete
vbdul> logminer xid 12345 recover delete
```

### PG DROP/TRUNCATE 恢复（PostgreSQL）

`db_type postgresql` 时，`scan filesystem free` 会额外生成 `dict/pg_drop_fragments.dul` 记录 PG 碎片，配合以下命令恢复：

| 命令 | 说明 |
|------|------|
| `list fragment` | 列出扫描到的 PG 碎片 |
| `show fragment <F001>` | 查看碎片及页面详情 |
| `match ddl <ddl_file>` | 用 DDL 文件匹配碎片，生成 table_ddl.dul |
| `extract pg fragment <FID> using ddl <ddl_file> [--toast <FID>] [--output <dir>]` | 提取指定碎片（含 TOAST） |
| `extract pg table <schema.table> using dict` | 按 col.dul 字典提取整表 |
| `extract pg table <schema.table> using ddl [path]` | 按 table_ddl.dul 提取整表 |

### License 管理

| 命令 | 说明 |
|------|------|
| `license request [db_type]` | 生成 license_request.txt（发给厂商签发） |
| `license show` | 查看授权状态（版本类型 + 主机指纹） |

Beta 版本无需 license，正式版需凭 request 文件签发授权。

## 字典文件

`unload dict` 自动从系统表生成以下字典文件到 `dict/` 目录：

| 文件 | 内容 |
|------|------|
| `db.dul` | 数据库信息 |
| `schema.dul` | Schema 列表 |
| `obj.dul` | 对象/表列表 |
| `col.dul` | 列定义 |
| `type.dul` | 类型定义 |
| `part.dul` | 分区定义 |

首次运行必须先执行 `unload dict`，后续操作依赖字典文件。

## 配置文件 (config.dul)

```
byte_order      little
block_size      8192
database        dultest
data_dir        /home/vastbase/data/vastbase
output_format   text
output_dir      data
extract_dir     extract_file
restore_dir     restore
db_version      v3
delimiter       |
charset         UTF8
ora_number      on
db_type         vastbase
wal_dir         /home/vastbase/data/vastbase/pg_xlog
backup_dir      /home/vastbase/backup
unload_dict_deleted off
unload_deleted  off
```

## 编译

```bash
mkdir build && cd build
cmake ..
make
```

产物为静态链接可执行文件，无外部依赖，可部署到任意 Linux x86_64 服务器。

## 打包

```bash
./package.sh
```

产物位于 `packages/vbdul-Beta_1.0.0-x86_64.tar.gz`。

## 支持平台

| 平台 | 状态 |
|------|------|
| Linux x86_64 | 已支持 |
| Linux aarch64 | 已支持 |

## 注意事项

- vbdul 以只读方式访问设备和数据文件，不会修改原始数据
- 所有输出写入 `data/`、`dict/`、`extract_file/` 目录
- 操作前建议先备份原始数据文件
- 仅支持 Heap（行存）引擎，ustore（undo 引擎）和 cstore（列存引擎）暂不支持
