# Changelog

## v1.0.0 (2026-06-26)

首个正式版本。

### 核心恢复引擎

- 字典自动构建（`unload dict`）：直接从数据文件解析系统表，无需数据库在线
- Heap 页面精确解析：Tuple 字段解码、MVCC 事务可见性判断
- 大字段支持：TOAST 内联 PGLZ 压缩 + 外部 chunk 存储（1MB+ 实测完整恢复）
- 复杂类型：JSONB（JB 二进制格式）、数组（int[]/text[]/bool[]/numeric[]）、number（ora_number 兼容）
- 分区表：pg_partition 字典解析，`unload table ... partition <name>` 按分区导出
- 无字典模式：`unload table <filenode_id> column <types>` 仅凭 OID + 类型导出
- 离线校验：`verify table` 数据一致性检查
- 导入配套：`imp` 自动生成 psql 导入脚本

### 数据库支持

- VastBase 全版本：v0（PG11 内核）/ v1（PG14）/ v2（PG16）/ v3（openGauss 6.0）/ v5（PG17）
- PostgreSQL 11 ~ 17（`db_type postgresql`）
- `db_type` + `db_version` 双参数分派独立解析路径

### 文件系统级恢复（数据库无法启动 / 文件被删）

- **ext4**：inode 扫描、extent tree 遍历、空闲块扫描、删除时间过滤提取
- **XFS**：AGFL 环形队列扫描、BNO/CNT B+tree 遍历、RMAP 反向映射、已删除目录项解析、inode 扫描、空闲碎片扫描
- **LVM**：VG/PV/LV 自动识别，多设备聚合扫描
- **rm -rf 数据目录**：`scan fs datadir` 按目录树定位 + `extract fs datadir` 提取到 restore/
- 多级 B+tree 遍历、页面指纹分组、in-use bitmap 排除

### DROP TABLE / TRUNCATE 恢复（无备份）

- `scan filesystem free [parallel N]`：空闲空间扫描定位被释放数据页面（ext4 空闲块 / XFS 空闲碎片）
- `list free page ext col <n> [tlen <m>]`：按字段数/tuple 长度过滤缩小范围
- `extract free page using vb_ddl`：DDL 匹配提取 + 自动 TOAST 关联
- `unload table <name> recover`：一步导出为可导入 SQL

### ProBackup 集成

- `list backup` / `list archivelog` / `show backup <id>`：backup.control 解析、WAL 区间合并
- `unload dict backup [BACKUP_ID]`：从备份集构建字典
- `restore table`：从备份还原单表数据文件
- `recover table ... stop_at lsn|xid`：选择性恢复到指定 LSN/XID

### Logminer（WAL 误操作回滚）

- `logminer <start> <end>`：挖掘 WAL 中的 DML（VastBase/PostgreSQL 自适应）
- 过滤：按表 / XID / LSN 区间 / 提交时间
- `logminer xid <XID> recover`：按事务反向恢复——DELETE 生成数据文件、UPDATE 生成反向 UPDATE SQL
- 大事务缓冲恢复（`nobuf` 直写模式）

### 授权与工程化

- Ed25519 签名 license 授权体系：`license request` 生成机器指纹申请文件，离线签发
- 静态编译 + UPX 压缩，零运行时依赖；Linux x86_64 / aarch64
- 坏块容错：损坏页面、I/O 报错自动跳过
- 只读安全：所有设备/文件以只读方式访问，输出仅写入独立目录
