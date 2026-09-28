# Changelog

## v1.0.1 (2026-09-28)

### 修复

- **★ License 绕过漏洞（Release 版，自 v1.0.0 起存在）**：`src/repl.c` 里 license 校验被套在
  `if (ctx->config_loaded)` 中，于是**只要没有 `config.dul`**（删掉它、或把二进制换到别的目录运行），
  整段校验就被跳过 —— Release 版无需任何 license 即可使用全部功能（`unload dict` / `unload table` /
  `recover` / `logminer` 全部照跑）。现改为**无条件校验**：`config.dul` 所在目录只用于**定位** `license.dul`，
  找不到 config 时回落到可执行文件所在目录，再回落 `.`。失败仍进受限模式（`license request` 逃生通道保留）。

- **`unload dict` 报 `Invalid num_mappings: 63 (max: 62)`**（`src/dictionary/filenode_map.c`）
  `pg_filenode.map` 解析把老格式的上限写死成 62，导致合法 map 被拒。实际各内核上限不同：
  PG ≤15 / openGauss 老格式 = 62（512 B）、**VastBase G100 2.2.5 = 63（520 B，客户现场命中）**、
  PG 16+ / v2,v5 = 64（524 B）、openGauss 4KB = 510（4096 B）。现老格式统一按 **64** 放行。
  报错信息补充 `file_size` / `format` 便于定位。
  ⚠ 该错误在 `unload dict` 中**不中断**（只 WARN 并退回 `filenode=relid` 兜底），
  后果是整份 global map 被丢弃、系统表 filenode 全错（实测 pg_database 变成 `1262→1262`，
  真值 `1262→15984`）——遇到此错修复后**必须重跑**。

- **字典文件空字段解析**（`filesystem/disk/vg/pv/lv` 共 7 处读，新增 `util/strutil.c: vbdul_split_pipe`）

- **XFS B+tree 内部节点指针偏移**：ptrs 起始位置按 `maxrecs` 计算、指针个数用 `numrecs`
  （原先按 `numrecs+1` 读，导致 `Invalid btree magic 0x0` 且静默返回 0 条）；
  含 3 处递归共用 buffer、is_v5 误判一并修复。204 真 XFS 实测 1505 条与 `xfs_db` ground truth 对齐。

- **aarch64 包携带工具链指纹**（`package.sh`）：宿主 `strip` 无法处理交叉编译的 aarch64 ELF
  （`Unable to recognise the format of the input file`），而该行以 `|| true` 结尾 ⇒ 失败被静默吞掉，
  arm64 包一直带着 `GCC: (...)` 指纹发出去（x86 因宿主即本架构所以一直是干净的）。
  现按架构选用 `aarch64-linux-gnu-strip/objcopy`；又因 upx 4.2.2 无法压缩被删过节区的 aarch64 ELF
  （`CantPackException: xspan unexpected NULL pointer`），改为 `objcopy --update-section .comment=/dev/null`
  抹空内容（保留节头）。
- **打包流程加硬门**：upx 缺失 / UPX 打包失败 / 工具链指纹残留 ⇒ 直接报错退出，不再静默产出
  未压缩或带指纹的包。

### 变更

- **Beta / 发布版标志分离**：版本串不再硬编码 `"Beta x.y.z"`（旧版发布版会显示 `Beta 1.0.0 (Release)`，
  两个标志互相矛盾、无法辨别包类型）。现版本串只含版本号（`1.0.1`），由 `VBDUL_BETA` 在 banner 与
  `--version` 里追加标志：Beta 构建显示 `1.0.1 (BETA)`，发布构建显示 `1.0.1 (Release)`；
  版本串从 `VBDUL_VERSION_*` 自动拼接，避免手工漂移。

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
