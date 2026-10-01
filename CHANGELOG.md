# Changelog

## v1.1.2 (2026-10-02)

热修版：复核抓出的两个真缺陷 —— v1.1.1 引入的 BTREE fork 崩溃路径，与
TOAST 误检修复当时漏数的两处扫描点。

### 修复

- **★`scan filesystem` 崩溃（BTREE fork NULL 解引用）**：v1.1.1 补的
  bmap btree fork 真遍历在 `xfs_parse_inode_extents_ex` 的 BTREE 分支
  无条件解引用 ctx，而兼容入口（`xfs_parse_inode_extents`）恒传 NULL ——
  损坏/已删的 BTREE 格式目录 inode 走目录扫描两个 fallback 路径时直接
  SIGSEGV。无 ctx 时改为优雅返回空 extents（并补上已分配缓冲的 free）。

- **★TOAST 指针裸扫误检（v1.1.1 漏数的 2/4）**：v1.1.1 判定收敛时清点
  "全库共 2 处裸扫"，实际 `extract inode time ddl` 与
  `extract free page ext ddl` 内还有两处 `01 12` 弱校验循环 —— 小端
  int4 值 0x??1201（如 4609）叠加后续字节可同时满足旧校验的
  relid/valueid>16384、extsize>0、rawsize>0，注册假 TOAST 指针导致后续
  提取找错文件。统一改走共享 `vbdul_va_is_external_ondisk()`，两条命令
  真/假指针对照样例实测：旧版各误检 1 个，新版零误检，真阳性零丢失。

- `scan filesystem` 活集行标签 `active` → `allocated`（该集合即 inobt
  权威遍历的已分配 inode，语义修正）。

## v1.1.1 (2026-10-01)

修复版：XFS 恢复实测（172.16.53.156 全链验证）暴露的一批真缺陷。

### 修复

- **★TOAST 指针裸扫误检吞行**：表导出/恢复路径用裸字节扫描 `01 12` 识别
  外部 TOAST 指针，无字段边界感知 —— 行首 int4 字段值形如 0x??0112 的行
  （实测 id=4609/70145/1053185/1118721 全中招）被误分流甚至彻底丢失。
  判定收敛为共享 `vbdul_va_is_external_ondisk()`（4 对齐 + rawsize/extsize/
  valueid 结构不变量），两处扫描点统一走同一实现。

- **★XFS `scan filesystem`：drop 后元数据未落盘导致漏扫**：XFS unlink 不
  释放 extents（挂在 AGI unlinked 链上推迟到 inode 回收），`sync()` 不触发
  回收 —— 磁盘 bnobt/AGI/inode fork 长期旧态，扫描实测只覆盖 10%。
  现在 `unload dict` 在写 XFS 元数据快照**之前**自动 sync + drop_caches +
  sync（需 root；非 root 打警告），实测覆盖 10% → 100%。

- **XFS 并发扫描互删临时文件**：两个 vbdul 同时扫同一 dict 目录时，一方
  收尾 remove 共享 `.xfs_free_ext_single.tmp`，另一方 `system(sort)` 随即
  读不到（`sort: cannot read` → 退化 in-memory merge）。tmp 文件名加 PID
  隔离，双进程并发实测零报错。

- **`xfs_free_ext.dul` 打印语义**：原先打印的是 VB 页计数却标 "rows"，
  现在打印文件真实行数（合并组数）与页数两个字段：`56 rows (78675 pages)`。

- **`unload dict` 全库被排除时假成功**：pg_database 有库但全部被
  `db_all_exclude`/模板跳过时，此前打印 "built for 0 database(s)" 返回 0；
  现在明确报错（点名库数与排除名单）非 0 退出。

- **BTREE fork 解析**：XFS inode 的 bmap btree fork 此前 "not implemented"
  直接返回空（大表 inode 全部丢失），补上真实 btree 遍历（BMDR ptrs 按
  maxrecs 偏移，复用目录扫描路径的收集器）。

## v1.1.0 (2026-09-30)

本版本主体是**多库支持**：一个实例几十个库一次处理，`unload dict` 建全部库字典，
凡涉及库内对象的命令加 `db <name>` 子句；同时并入 **db_version v6（VastBase G100
2.2.5）** 版本档与自定义表空间定位，并修复 v1.0.1 的 `imp` 必崩 SIGSEGV 等问题。

### 新增 — 多库支持

- **字典多库布局**：`config.dul` 的 `database` 留空即进入多库模式，`unload dict`
  一次循环构建全部库的字典——库级字典按 `dict/<dboid>/*.dul` 分库存放，顶层只放
  共享/实例级字典（`db.dul`、`disk.dul`、`ext4_*.dul` 等）；文件格式一个字节未改。
  `database` 有值则完全保持 v1.0.1 单库行为（平铺布局、命令可不带 `db`）。
  单库失败可见化（`obj.dul` 哨兵机制，跑前清、跑后验），不再假报完成。

- **`db <name>` 子句**（多库模式）覆盖全部库内对象命令：`desc` / `list schema` /
  `list table <schema>` / `unload table` / `unload schema` / `verify table` /
  `dump filenodeid` / `imp` / `restore` / `recover table` / `logminer`。
  子句解析在分词前统一剥离，按命令白名单门控、引号感知。

- **`list db`**（`list database` 别名）：列出实例内全部数据库。

- **`unload db <spec> [meta|data]`**：批量导出整库/多库。spec 支持
  `all`（减去排除名单，执行前列清单要 `y` 确认）、`a,b,c` 显式列表（不受名单影响）、
  `<spec> except x,y`。`meta` 只导 DDL，`data` 只导数据。输出为 expdp 风格
  （库头 + schema 缩进 + 每表 `DDL:`/`. . exported` 两行 + 收尾统计）。

- **输出文件 `<db>_` 前缀**：多库模式下导出文件统一带库名前缀
  （`mydb_public_orders.txt`），避免多库互相覆盖；单库模式保持 v1.0.1 原文件名。

- **`db_all_exclude` 配置**：`unload dict` / `unload db all` / `list schema|table db`
  跳过的库名单（默认 `template0,template1,template2,postgres,vastbase,atlasdb`），
  显式点名不受影响。

- **`unload dict using db.dul` 引导路径**：`global/pg_filenode.map` 定位不到
  `pg_database` 时手写 `db.dul` 当种子再解析各库——与 `config.database` 无关，
  两种 dict 布局都能跑（原先 `config.database` 非空的硬要求是 bug，已去除）。

### 新增 — 版本与表空间

- **db_version v6 = VastBase G100 2.2.5 版本档**：独立 catalog 表集（44 表）与
  type_map（2.2.5 无 oradate，`date` 回 4 字节 OID1082；name 列 64 字节）。
  未知 `db_version` 从静默回落 v3 改为响亮报错退出。

- **自定义表空间定位**：表不在 `pg_default` 时按 `base/<dboid>/<filenode>` →
  `pg_tblspc/<tsoid>` 软链接多路径解析；定位不到时清单式报出尝试过的路径。
  heap 数据文件缺失从**静默成功（0 行）**改为响亮失败，批量导出继续其余表并在
  收尾行计数（`N tables exported, M failed`）。

### 修复

- **`imp` 一执行就 SIGSEGV（v1.0.1 起存在，本版必修）**：`VbdulImportEntry×4096`
  的三个 ≈9.9MB 栈数组远超 8MB 线程栈，`imp` 任何形态（`all`/`schema`/`table`）
  必崩。改堆分配 + OOM 检查。

- **`unload dict` 全部库被排除时假成功**：pg_database 有库但全被
  `db_all_exclude`/模板跳过时，此前打 `Dictionary built for 0 database(s)` 返回
  0；现明确报错（点名库数与排除名单）非 0 退出。

- **logminer XID 恢复输出判别误吞 WAL 段名**：判据"首段全数字"把 24 位十六进制
  WAL 段名也当 XID（如 `000000010000001300000010`），原始输出被静默跳过。补
  十进制位数上界（≤10 位）。

- **pg_logminer 跨库 WAL 混入**：过滤用 `first_db_oid`（多库模式下恒 0）导致
  其他库的 WAL 记录混进输出，改为按当前选中库的 `ctx->db_oid` 过滤。

- **display/导入路径旧账**：4 个 display 函数 `malloc` 无 NULL 检查、
  `type_display[64]` 无界 `strcpy`、`desc_backup` 的 `strncpy` 不补 NUL——全部加固。

- **pg_partition 解析绕过版本分派**：回调硬编码 v3 列下标，v6 等其它版本档
  会读错列；改为走版本分派 + 按列名取字段。`pg_drop.h` 把"vastbase 且非 v3"
  误分类为 PG 内核（v6 会拿到 24 字节 PG 页头）一并修正。

- **`run()` 双 `fclose` UB** 与导出 `.sql` 无 `ferror` 检查：补 NULL 复位与写错检查。

- **命令失败映射进程退出码**：EOF 返回最后一条真实命令的状态（`exit`/`quit`
  不抹状态），未知命令/被 license 拦截 = 非 0；`license show` 恒 0。便于脚本化。

### 变更

- **`unload dict` 输出调整**：`template0`/`template1` 静默跳过建字典（`db.dul`
  仍列出）；每库开头一行 `unload db <名> (oid <oid>) dict:`；文件系统级字典行数块
  加头 `unload filesystem meta:`。

- **`utl_file_dir` 表全局排除**（VB utl_file 兼容包自建表）：`unload schema` 数据
  循环、DDL 枚举、`unload db` 清单计数三个口径一致排除；显式 `unload table` 点名
  仍可导出。

- **`unload db` 清单只计表数**，不再带数据量预估。

### 兼容性

- 单库模式（`config.dul` 的 `database` 有值）行为与 v1.0.1 逐字节对齐
  （204 三环境 A/B 实测：v0/v3/v5 字典与导出文件 md5 全等）。
- license 校验逻辑与 v1.0.1 一致（版本升级不作废 license；仍为无条件校验）。

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
