# Changelog

## v1.1.17 (2026-10-26)

损坏数据下的稳定性、jsonb 导出的正确性，以及索引 DDL 与可见性判断的修复。

### 内存安全

- WAL 回放时校验行指针槽号上界，损坏的 WAL 记录直接跳过，不再越界写内存。
- logminer 生成 UPDATE 语句时，缺失列按 NULL 输出，不再对未初始化的列值取长度。
- `list filesystem` 解析字典行时给字段扫描加宽度限定，首字段过长不再溢出栈缓冲。
- extract 的六处采样循环补上行指针边界检查，坏指针不再读到页外。
- `list table`（旧布局）与 `unload schema meta/data` 修正释放顺序，schema 名不再乱码。
- jsonb 数组与对象的收尾符号、分隔符在缓冲区末尾会越界写 1~3 字节（PostgreSQL 与 VastBase 两种内核共四处），已修正。
- VastBase 内核的 jsonb 容器偏移与长度增加数据区校验，元素数量的 32 位回绕一并检查，撕裂页不再导致越界读或超大分配。
- 元数据解析加固：ext4 extent 内部节点条目数、目录 inode 根条目数、XFS BNO 叶记录数按块容量钳制；LVM 超长设备名不再越界写；函数参数表拼接改为有界追加。

### 数据正确性

- jsonb 字符串含引号、反斜杠或控制字符时按 JSON 转义规则输出。此前按原文导出，重载会报错。

### 命令与输出

- 命令带前导空白时参数定位错位，粘贴的脚本行不再被误报为未知命令。
- `verify` 按内核分派：校验和算法仅对 VastBase 内核生效，PostgreSQL 内核不再全页误报；40 字节页头专属字段不再在 24 字节页头的引擎上误读。
- heap recover 逐列回显与日志挖掘缓存行的调试输出改为 debug 级别，默认不打印。

### 可见性与 DDL

- 离线扫描改为按提交日志判死：VastBase 参考 pg_csnlog（主）、pg_clog（辅），PostgreSQL 参考 pg_xact。此前只看元组头提示位，提示位未盖章的死版本会被当作有效行——导入脚本执行后立即卸载字典时，obj.dul 会出现同一 oid 的多个重复行并可能带出陈旧对象引用；业务表卸载没有可归并键，死版本与已回滚行会直接进入导出数据。提交日志不可读（残缺目录、裸盘、进行中事务）时维持原乐观语义，行为与旧版一致。
- obj.dul 写端按 oid 合并同 oid 的多个死版本行（取带 TOAST 的最新版），作为提交日志不可读场景的兜底。
- 锁持有判定按内核定义分引擎实现：VastBase 的"仅上锁"标记位于 t_infomask2（0x0800），此前按 PostgreSQL 的位置（t_infomask 0x0080）读取，`SELECT FOR UPDATE` 锁过且锁事务已提交的行会被误判为已删除、整行不导出；PostgreSQL 侧补上"仅置排他锁、无锁标记"的旧布局判定。
- 提交状态判定与内核对齐：VastBase 的 pg_csnlog 中"提交中"（1<<62）与"子事务"（1<<63|父 xid）两类条目不按已提交处理，子事务沿父事务链上溯；冻结元组（COMMITTED|INVALID）直接视为可见，不再回查日志。两本日志以终态为准——任一本给出已提交或已回滚即采用，未决不得否决终态。
- 多事务（MultiXact）删除的旧版本按成员解出真实更新事务后再判死活，VastBase 与 PostgreSQL 各自实现；PostgreSQL 的子事务按 pg_subtrans 追父事务状态。
- 数据卸载时，提交状态无法确定（提交日志未落盘或缺失）的行按乐观语义保留，并在该表汇总后给出提示与行数，不再静默多算。
- 函数索引的 DDL 改为渲染正常 SQL 表达式（函数、常量、算符、类型转换、COALESCE、LIKE 及嵌套），此前把 pg_node_tree 原文拼进 CREATE INDEX 且多个表达式位重复拼接；部分索引的 WHERE 谓词此前整条丢失（恢复后变成全量索引），现按同引擎渲染输出。
- 多列索引的 indkey 逗号分隔此前只解析出第一列；分区本地索引附加的系统列（tableoid 等负号列）不再渲染成问号，与内核定义一致。
- `dump table` 找不到数据文件时同时打印 OID 与 relfilenode（定位键实为 relfilenode）；`unload table` 数据文件缺失的报错附带定位提示，覆盖数据目录完整性、表空间挂载、字典与盘面错位三个方向。

### 索引 DDL 渲染

- 函数索引的表达式改为按顶层节点分别渲染左右操作数。此前先把参数拼成字符串、再按第一个逗号切分，操作数本身带逗号时（如 `(concat_ws('-', a, b) || 'x')`）整个表达式被切碎，导出的建索引语句无法执行。
- 表达式位超过 8 个时，此前会读到数组之外；现按实际槽位数截断，超出的位置输出 `/*expression*/` 占位。
- 参数拼接改为逐个写入并显式校验剩余容量，缓冲区边界处不再越界写；参数超过 16 个时同样输出占位，不再静默少写。
- 表达式树解析加递归深度上限，损坏的字典不会耗尽栈。

## v1.1.16 (2026-10-22)

ext4 目录树的死循环与内存占用、外部表空间支持、PostgreSQL 约束解析。

### ext4 目录树

- 目录项雕刻此前只过滤 `.` 漏了 `..`，根目录经 `..` 自指被反复入队，扫描不终止并持续占用内存。现每个目录只访问一次。
- 目录树改为流式处理，常驻内存与文件总数无关。
- 支持按 rm_dir 剪枝，只遍历恢复目标路径内的子树。
- 修复 ext4 设备误做 XFS 探测产生的告警噪音，以及 XFS 超级块 magic 不符时的错误文案。

### 稳定性

- jbd2 尺寸表插入只设 inode 号，代数比较读了未初始化内存，ext4_logsize.dul 内容随机漂移并影响恢复文件尺寸；已修正。
- 分配与失败路径加固：目录队列 realloc 半失败、jbd2 数组写空指针、mmap 失败未检测、磁盘满写截断、路径表分配无检查等，极端情况从随机崩溃改为明确报错。
- ext4 超级块几何不再免检（非法块大小、组度量会除零，描述符尺寸会溢出栈），改为拒识；XFS 大目录块跨批越界读补齐重读。
- ext4 目录树提取分支此前被 XFS 前置检查挡死，现已打通；提取从扫描期页快照重建，扫描后块被复用不影响结果。
- 线程创建失败从静默继续改为中止；`parallel N` 非法值给出提示，串行与并行产物一致。
- `scan_fresh` 同时清目录树游标，避免陈旧游标导致漏扫。
- 扫描时缺 `ext4_free.dul` 则现场读块组位图生成，纯镜像（无 data_dir）工作流可用。
- 路径读写宽度对齐、1K 块组描述符定位公式统一、空轮不截断上一轮产物、若干文件描述符泄漏。

### 外部表空间

- `unload dict` 支持建在外部表空间中的数据库（系统 catalog 与 pg_filenode.map 物理位于表空间内），自动生成 `tablespace.dul`。
- `pg_class.reltablespace=0` 且 base/ 未命中时按 pg_tblspc 扫描兜底，单表卸载、TOAST、校验等调用点全部生效。
- `unload table` 的 .sql 从裸 CREATE TABLE 升级为完整 DDL：列默认值、PK/UNIQUE/FK/CHECK 约束、索引、触发器、自有序列；序列参数按内核布局读取。
- 新增 PG10~17 pg_tablespace 解析。

### PostgreSQL 约束与解析器

- PG16+ 把 pg_constraint.coninhcount 由 int32 缩为 int16，字段表未按版本拆分，导致约束列清单输出空括号；已按 PG15/16/17 重新拆分。
- inet / cidr 越界读加固。
- `unload table` 尾部未知词从静默忽略改为报错。
- 活库检测进程名补充 vastbase；术语统一为 PostgreSQL / VastBase；导入脚本统一使用 psql。

### 扫描输出与导入脚本

- `scan filesystem` 输出改为 inode、空闲空间、目录树三段式，参数行合并，各段带计数汇总。
- 空闲空间扫描每 10GB 输出一次进度（串行、并行都有）；XFS 并行路径补收尾汇总，进度不再超过 100%。
- 无 extents 标志的 inode（符号链接、设备文件等）不再逐条告警；`extract fs datadir` 逐文件明细转入 debug，错误聚合到收尾汇总。
- 导入脚本默认库名改为导出时选中的库（此前硬编码测试库名）；psql 只带 `-d`；库名做 shell 转义。
- 多线程进度计数改为原子操作。

## v1.1.15 (2026-10-20)

畸形输入的健壮性加固，正常数据的输出不变。

### 修复

- **jsonb 值长度收口**：值长度与偏移来自磁盘，撕裂页或类型错配时可越过缓冲区末端；字符串、数值、嵌套容器分支统一按数据区剩余字节收口。
- **jsonb 值缓冲防超大分配**：值长度按相邻 offset 差值推导，无序 offset 会使其回绕成 GB 级；缓冲上界改为数据区剩余字节。
- **numeric 的 varlena 头边界**：4 字节头读取补长度检查，总长字段按实际剩余收口（jsonb 内嵌与独立 numeric 列两条路径）。
- **压缩 varlena 判定门槛**：旧门槛只保住头本身，4~7 字节残尾声称压缩值时越界读；门槛提高到 8。
- **NULL 位图容纳校验**：元组头声称的列数可达 2047，旧实现不校验 t_hoff 是否容纳整个位图，把非元组字节当元组解析（free 页扫描、DROP 恢复）时位图越界读；按 heap_deform 不变式补双侧校验。
- **numeric Long 布局残尾守卫**（VastBase 侧）：2~3 字节残尾读 weight 越界，补齐与 PostgreSQL 侧同款守卫。

### 内存

- **字典上下文深释放**：旧实现只拆哈希节点不释放值对象与列数组，每次换库重装字典泄漏一整份（数十 MB 级）；共享指针哈希保持只拆不释放，杜绝 double-free。

### 工程

- 新增发布前自动化门禁：全仓 AddressSanitizer / UBSan 构建、历史缺陷回归用例、双引擎类型渲染模糊测试，并接入打包脚本。

## v1.1.14 (2026-10-19)

字典与 DDL 导出、扫描与恢复的缺陷修复。

### 字典与 DDL 导出

- **PG12~17 pg_proc 补 oid 首列**：PG12 起 catalog 的 oid 从系统列改为显式第一列，旧定义从 proname 起读、全字段左移错位——proc.dul 函数名乱码、oid 全 0、用户函数整体不可见；PG10/11 定义不变。
- **pg_am / pg_rewrite 补 PG12+ oid 回退**：新布局下 am.dul / rewrite.dul 的 oid 恒 0，访问方法（am）与重写规则（rewrite）反查永远落空。
- **索引 DDL 的 USING 取索引自身 relam**：旧实现用堆表 relam 反查，PG12+ 与 VastBase 下 heap 命中时导出非法的 `USING heap`，未命中时 hash/gin/brin/gist 全部塌成默认 btree。
- **触发器 DDL 的 tgtype 位义修正**：ROW/BEFORE/INSTEAD 位全错，普通 AFTER INSERT FOR EACH ROW 被导成仅视图合法的 INSTEAD OF ... STATEMENT（建表必失败），BEFORE 从不输出；触发器函数名改按 tgfoid 反查。
- **part.dul 短行守卫前移**：损坏行或超长行被劈出的短尾段在字段解析前先守卫，不再崩溃，跳行并告警继续。

### 扫描与恢复

- **XFS 断点续扫恢复 free ∪ unlinked 并集**：resume 路径只按裸 free 列表切片，AGI unlinked 链上的块（rm/DROP 后仍挂链）零覆盖、提示消失；与 fallback 路径统一为公共实现。
- **XFS 磁盘检查点会话信任门 + 多盘游标隔离**：崩溃残留的扫描游标在重跑时被无条件采纳，新列表前 K 个 extent 永不扫、旧 part 文件混进收尾；多块 XFS 裸盘游标互踩。现仅首个可扫设备采纳游标，信任失配先清检查点。
- **XFS 并集区间减法去重**：free 与 unlinked 两表在部分释放或崩溃窗口可重叠，不去重会同段双扫、恢复出重复元组；实现区间减法（全包含删除、中间挖洞拆两段、边缘裁剪）。
- **ext4 GDT 基块错位修复**：4K 块设备组描述符基块算式把 boot 与超级块区的块 0 当 GDT 读成垃圾，inode table 定位整支失效、journal 解析恒空——rm-rf 恢复的文件长度退化为块数猜测。改从正确基块起算。
- **obj.dul 同 oid 合并补 relnode 重插**：VACUUM FULL / CLUSTER 重写表后的新 relfilenode 查不到——unload table 有数据，而 logminer 与 WAL 回放按新 node 解析的 DML 静默丢失；合并处按新 node 补插哈希。
- **批读短读消费循环越界**：活库卸载中数据文件收缩（或坏道恰好落在批边界）时，消费循环按计划页数迭代到陈旧缓冲，上一批残页被再解析、重复行进 .txt；循环界改为实际读取页数。单页 fallback 同步改紧凑填充（旧写法批中间坏一页时每批永久丢尾部好页）。
- **区间减法元素计数越界**：区间中间挖洞拆分后有效元素恰为 np+1，多记 1 使消费循环越界读堆尾；已修正并补中间挖洞用例。
- **extract free page 零 DDL 回退补库解析**：配置了 database 时，`extract free page using vb_ddl|pg_ddl`（不带 db 子句、无手写 table_ddl.dul 的 TRUNCATE 场景）此前恒报 "no database selected"，同会话先跑 unload dict 也不生效。补齐后手册承诺成立：配置 database 即可省略 db 子句；db 子句与 config 同库放行、异库明确拒绝，不静默换库。
- **`extract free page ext <N,N,...>` 定点提取接通**：手册记载的指定碎片提取形态此前在命令分派层不可达（恒 Unknown command），本版接通。
- **XFS 目录树扫描父节点反查改为哈希**：汇总段对每个路径线性扫全表反查父 inode，数十万路径下纯 CPU 空转数十分钟形同挂死；改为路径哈希反查后同场景完整跑通，产物与旧版逐字节一致。

### 改进

- **extract 期覆写复验（页指纹）**：扫描为每个确认的数据库页记录 LSN 与整页指纹，extract 复读比对；扫描到抽取之间被复用的页自动跳过并计数告警，不静默导出他人数据。指纹文件缺失（旧版扫描产物）提示后放行，向后兼容。
- 移除零读取的调试遗留产物 ext4_free_ext_single.dul，每次扫描不再白写一份逐页文件。
- `extract xfs` / `extract ext4` 遗留空壳拆除（分派层直接拒绝并指引现行命令）；退役子动词的提示与 .dul 头统一指向 `scan filesystem`；logminer help 补 `[db <name>]` 选项行。
- obj 加载注释记录 UNLOGGED 边界：UNLOGGED 语义在 relpersistence，活库 CHECKPOINT 后页仍可不落盘，导出 0 行属正常现象。

## v1.1.13 (2026-10-16)

logminer 的 LSN 与起止参数修正，free 扫描的并行重切。

### logminer

- **VastBase 侧 LSN 会计修复**：记录 LSN 误把映射文件内偏移加到页面绝对 LSN 上（页偏移算两遍），段内越往后 LSN 越虚高，start_lsn / stop_lsn 过滤随之失准；PostgreSQL 侧原本正确。
- **UPDATE / DELETE 不再整条跳过**：旧逻辑"主数据无 OLD/NEW 标志即跳过"会把正常记录全误丢（新元组在 block data 不在主数据）。现 REDO 照常输出（UPDATE 带新值、DELETE 到表名），UNDO 列标注旧行镜像不在 WAL——VastBase 引擎即使 wal_level=logical 也不写旧行镜像。
- **起止参数自动识别 LSN**：传 X/Y 即自动换算所在 WAL 段（timeline 从 wal_dir 反查）并设精确 LSN 区间过滤；只挖一个段则起止同名。VastBase / PostgreSQL 双侧一致。
- **段名报错给格式指引**：补 24 位示例、ls 提示与 start_lsn / stop_lsn 指引；修 TLI 反查的数值溢出。
- **xid recover 诚实计数**：无旧行镜像的匹配行报 "matched but not recoverable"，不再静默显示 Matched 0。

### free 扫描并行重切（XFS / ext4）

- **按大小切工作单元 + 贪心装载**：旧实现按 extent 个数均分且线程数被 extent 数封顶——大盘 DROP 大表后空闲区是少数巨型 extent 加碎片，单线程独扛最大 extent、其余线程干等；单个巨型 extent 配 `parallel N` 还会被串行守卫打成单线程。现切成约 1GB 的页对齐工作单元，按大小降序分配，单数组摊平。
- **切点重叠 page_span-1 块**：滑窗起点盲区深度恰为 page_span-1（页头落在单元末尾 page_span-1 块内的页，8K 窗口伸不出单元边界）。旧版重叠 1 块在块 >= 8K 时重复检出同一页、块 < 4K 时仍漏检；现回退 page_span-1 块起扫，两单元窗口首尾相接，不重不漏；chunk 下限加固防死循环。
- ext4 预存问题一并修复：free 线程 realloc 失败直接置 NULL 导致 merge 解引用；补缺失的 pthread.h。
- **XFS 批尾双跳与批缝盲区**：修复批尾偏移跳两遍（下一批整批跳过，块 <= 2048 时约半数窗口起点从未尝试）与批间重叠不足，pthread_create 返回值补检查。

## v1.1.12 (2026-10-15)

recover 产物命名统一，imp 只认 unload 产物的安全版。

### 修复 — imp

- **help 语法回真**：`recover table` 实际语法是 `until lsn/time/xid`，help 误写成不存在的 `from backup <id> … stop_at` 组合；`restore table` 补可选 `from backup <id>`。
- **recover / logminer 族产物名统一带 `<db>_` 前缀**：单库模式下 recover 产物此前不带库名前缀，现与 unload 同口径（recover_、pg_logminer_、logminer_<XID>_ 三族一并统一）。
- **imp 显式跳过 recover / logminer 族脚本**：recover 的 .sql 自带标记且与 .txt 成对落在 data/，此前会被 imp 当 unload 表误导入；xid recover 的 UPDATE .sql 会解析出虚假 schema 条目。名字以 recover 开头的真实表不受影响。

### 改进 — 字典与扫描

- **db_all_exclude 默认名单收窄**：template0/1/2/postgres/vastbase/atlasdb 六库改为仅三个模板库；vastbase 系统库开箱可抽（`unload table xxx db vastbase` 无需先改名单），被排除库不再打印空头跳过行。
- **XFS scan filesystem 输出降噪**：61 处内部进度行压至 debug 级，删除扫描横幅两行。
- **finobt 判定改读超级块特性位**：v4 与 mkfs finobt=0 的盘安静走 freelist，不再每 AG 一条误报；删除零调用的死代码。
- **Beta 试用期 30 → 90 天**：锚定编译日、防时钟回拨，只影响 Beta 构建，正式版无感。

## v1.1.11 (2026-10-11)

类型覆盖扩充与端到端恢复质量修复。

### 新增 — 类型覆盖

- **enum / composite（复合类型）**：pg_enum 字典两种布局（PG12+ 首列 oid 是数据列；VastBase 3 列 oid 在元组头）；复合类型迷你头按内核分派（PG 与 VastBase 各一版），composite 的 has_null 位图感知。
- **几何族 / tsvector / range**：point / path / polygon 等 6 类（含 polygon boundbox 环绕）、range 3 类。
- **V0（AtlasDB / PG11）适配**：pg_enum 3 列与 name 长度 64（越界会导致 label 恒为 \N）、扩展日期 OID 数组元素渲染。

### 修复 — PostgreSQL 侧数据正确性

- **PG16 / PG12 / PG13 pg_attribute 列序对齐源码**：attnum、attlen 等列位错档，PG16 下 unload 出的列定义全错。
- datetime / infinity / BC 日期 / interval 精确重写；NaN / Inf float；单列空串；bit / mac / lsn / money；数组元素与多维对齐（packed 元素）；jsonb 嵌套与超长值。
- detoast 与解压值按类型分派渲染，jsonb、numeric、数组不再裸拷贝。

### 修复 — VastBase 侧数据正确性

- **大 numeric 4B 头误判**：头字节大于 1000 被当"无头"吃掉。
- jsonb 三重缓冲（1024 / 256 / 缺 NUL 终止）；空 bytea；压缩与 detoast 分派；money / mac / oradate 数组。

### 修复 — 扫描与恢复链

- 越界读写、uint32 回绕活锁（日志挖矿）、静默截断、几何错位。
- toast 页分类"整页同 chunk_id"误杀小值外部存储（EXTERNAL 大字段恢复全占位）。
- 可见性 FROZEN / LOCK_ONLY 分类（VastBase 与 PostgreSQL 双实现分派）。
- obj 同 oid 双 toast 关系合并改为后行者赢；schema 过滤 dropped 列。

## v1.1.10 (2026-10-08)

崩溃修复与硬编码审计修复。**建议 v1.1.8 / v1.1.9 用户立即升级**：这两版在"误删后立即 scan"（产品主场景）必崩。

### 修复 — 崩溃

- **日志挖链接传值当指针，扫描首阶段必崩**：两处调用把计数按值传给指针参数（0 被转成空指针），函数内自增即段错误。日志窗口内有目录块镜像就触发——rm-rf 后立即 scan 的整条链路断在这里。

### 修复 — rm-rf 恢复正确性

- **rm 后 inode 复用过滤**：恢复出的文件可能是 rm 之后活实例新建的文件（allocator 复用释放的 inode）。现按 rm 子树 crtime 锚拒绝复用 inode，统计行恒显。

### 修复 — 硬编码审计

- **type_map 补系统复合类型与伪类型**（v3 / v6 / v0）：未知类型此前回退 varlena，导致定长列连锁错位。
- **catalog 字段槽 1024 → 8192**，三种解析器的顶格截断给出告警：长函数体、视图规则、索引表达式此前静默失真。
- **TOAST 指针收集的 256 硬顶（三处）改动态扩容**：满后静默丢大字段。
- **数组元素对齐取消猜测**：改按字典 typalign 与 type_map OID 兜底，timetz[]、macaddr[] 从第 2 个元素起错位的问题修复。
- **logminer DML 拼接的下溢越界**改为定值截断。

## v1.1.9 (2026-10-08)

unload / dict 与 scan（XFS、ext4）两条主线的缺陷审计修复，覆盖内存安全、静默失效、数据正确性与可移植性。

### 修复 — 字典读写契约

- **.dul 四个尾字段的读写契约从未接通**：函数体、列默认值、包体、触发器体此前静默丢失。
- **中间字段转义成组**：标识符或表达式含 `|`（如 `CHECK (a || b)`、表名 `t|b`）不再整行错位；写侧 19 处与读侧 25 处统一走转义正典，.dul 头部加格式代次标记。
- proc.dul 无 propackageid 档位只写 8 字段（读侧整体错位）、constraint.dul 差一位（CHECK 约束静默全丢）、index.dul 行缓冲装不下长表达式、dict_loader 三处解析缺陷。
- 分区四连修：V3 目录字段槽 128 → 1024（LIST 边界较长时变 `{}`）、pg_partition 硬顶丢分区、LIST 边界引号化两处栈溢出。

### 修复 — 数据正确性

- **字面 `\N` 数据值与 NULL 标记混淆**：值恰好为 `\N` 的列重载时读成 NULL（丢值）。现 NULL 判定上移、数据值统一转义，字面 `\N` 输出 `\\N`，与 PostgreSQL COPY text 口径一致，空串 / NULL / 字面 `\N` 三态可区分。
- **冻结元组被当"插入回滚"丢弃**：VACUUM 过的表静默少行。
- **空串列静默重复上一列的值**。
- varlena 长度助手不认 18 字节外部 TOAST 指针（被 DROP 的列之后整行错位）。
- 内联压缩 varlena 的 PGLZ 流偏移错 4 字节；bytea 渲染三处。
- **numeric 精度**：weight < -1 时小数前导零 digit 组未补（`-0.000042` 渲染成 `-.42`，小数值虚大 10^4n 倍）；NumericShort 的 weight 符号幅值误读（-2 读成 -62），共 5 处实现。
- PostgreSQL 侧 jsonb `count` 与数组 `dim1` 越界读。
- imp 文件名反解：schema 名含 `_` 时丢表或错 schema 灌数据。
- 数据 .txt 写失败（盘满）静默当成功。
- 表级 DDL 与分区键三处定长栈缓冲越界写。

### 修复 — 类型系统

- **type_map 按版本独立定型**：v0 = AtlasDB（date 8 字节 PostgreSQL 布局 + varchar2/nvarchar2）、v3 = VastBase（oradate 8 字节）、v5 = PG 17 标准（date 4 字节、无 Oracle 别名）。旧注释"v5 date=8B"是错的，一刀切写法已删。
- 未知 db_type 从静默回落 vastbase v3 改为显式报错。

### 修复 — XFS 与扫描

- **bmap btree 块头的 bb_level / bb_numrecs 读反**：BTREE 格式 inode 一条 extent 都收不回。
- **8 处 db_version 漏拷**：v3 实例整条页识别失效。
- AGI unlinked 收割漏拼 AG 分量（AG > 0 的 inode 号系统性偏小）。
- 多设备上下文别名 double-free / fclose、fds[] 零初始化当有效 fd、位图上界用错量。
- 残留 extent 扫描越过调用方缓冲、EXTENT 目录 inode 二级指针、日志解析的 ops_end 不夹缓冲与短读下溢死循环、free 扫把"页 = 2 块"写死（8K 以上块全错）、页缓冲按 block_size 分配、scan_stats 时间戳未初始化。

### 修复 — ext4

- 目录树 BFS 每条路径 free 两次（必然 double-free）、队列硬顶 1024 静默截断、inode-table 容量砍到 1/128（journal 挖矿静默失效）、残留 extent 字段偏移、固定 4096 字节缓冲装 block_size（大块栈溢出）、路径拼接的注入点。

### 修复 — 设备与备份

- 磁盘探测三处 double-fclose、XFS magic 缺字节序、NVMe 分区号恒 0 且漏扫、mounts 栈缓冲无边界。
- backup 字典：压缩页假读报成功改为跳过并告警；页号取块头而非顺序索引；5 处吞返回值改为检查。（pg_probackup 完整解压支持待真实环境，本版不含）

### 修复 — 可移植性

- switch 标号后紧跟声明，RHEL / CentOS 8（GCC 8.5）编译不过，全仓仅此一处。

## v1.1.8 (2026-10-06)

rm -rf 数据目录恢复：误删 data_dir（XFS / ext4）的整库文件级恢复链，外加断点续扫、设备直通、free 区间扫描、坏超级块回退与 ext4 目录树恢复。

### 新增

- **rm -rf 恢复链**：
  - 目录骨架重建：残余条目滑动雕刻、短文件名目录残余解析、孤儿子树嫁接与 `_lost` 聚簇、跨代残留按 crtime 与 LSN 过滤。
  - 日志挖掘：XFS 日志中的 inode 核心、短文件名链接、目录块碎片写入 `xfs_loglink.dul`，BFS 嫁接区按表补链。
  - 页雕刻兜底：inode 已彻底失效的文件按空闲页段裸输出。
  - 扫描时页快照：识别命中页边扫边存镜像，提取先快照后活盘，对"扫后块被复用"的窗口免疫。
  - 相位自适应：rm_dir 在场时目录骨架先于页扫（按易失性排序）。
  - 代际绑定硬门：无本代候选时拒绝旧代嫁接，输出 NOT FOUND 并给出原因；损失量化把 unlink 风暴 extent 分解为可恢复与已复用。
  - 首阶段收割 AGI unlinked 链（新鲜 DROP 的 inode 不在 finobt）。
  - 灾难引导：data_dir 指挂载点即可起步（unload dict 自举 filesystem.dul）。
- **运营工程**：
  - 会话持久化与断点续扫：scan_session.dul 分相位检查点，kill 后从游标接续。
  - 设备直通：`config device <path>` 直接读 dd 镜像、分区或 LV。
  - free 区间合流：树扫遍历 bnobt 与 unlinked 的并集区间，坏 btree 自动退化全盘段。
  - 坏超级块回退：主超级块损坏时从 AG 副本自洽恢复几何。
- **ext4 目录树恢复（第一层）**：inode 表驱动 BFS 加双路雕刻（残余名与空隙隐藏名），接入 extract 统一命令。
- **商业对标补缺**：reflink / COW 的 rmap 兜底（owner → extents 反查）、AGF 损坏时聚合 ABTB 叶块重建 free 表、XFS 与 ext4 日志挖链接。
- **命令面**：`scan filesystem [parallel N]` 全链总命令（页扫、inode、目录树等相位）；scan_stats.dul 把一次扫描的统计全部留现场。

### 修复

- **块号约定三连修**：bno 遍历起点按线性绝对块号统一（此前 AG1 以上双重偏移丢条目、AG3 全灭）。
- 候选嫁接选根反选（同代内按目录子项数与 crtime 最老定真根）、候选暂存重复入列挤掉真根。
- 提取器补建父目录：`_unnamed` / `_lost` 合成容器不再 ENOENT；静默失败计数全部补上原因。
- datadir 提取按文件系统块大小读页（BLCKSZ 是数据库页大小）。
- 断点续扫游标两处死代码（边界判定永不触发、活游标无读方）。
- TRUNCATE 恢复缺 table_ddl.dul 时回退字典形状（零 DDL 可跑）。
- 陈旧误报清理（多库模式 data_dir 提示等）。

## v1.1.7 (2026-10-03)

架构统一：VastBase 与 PostgreSQL 的 DROP / TRUNCATE 恢复链合一——同一管线、同一 .dul、同一命令面，删除 pg_drop 模块 5577 行冗余代码。

### 重大变更

- **统一恢复链**：VastBase（VastBase 内核）与 PG（社区内核）的 DROP / TRUNCATE 恢复走同一条 free-page 管线（scan filesystem → extract free page → unload table recover），产出同一份 `xfs_free_ext.dul`。页面与 Tuple 解析严格按内核分派（`pg_` / `vb_` 双轨，不混用）。
- **命令面统一**：
  - `extract free page using vb_ddl|pg_ddl`：token 与 config 的 db_type 交叉校验，配错明确拒绝，防跨内核混淆。
  - `extract inode <n>|all [time ...]`：裸语法；不匹配文件保留为 inode_N.dat，不再删除。
  - 移除 fragment 命令族（list / show / match / extract fragment），后续作为统一增强重做。
- **pg_drop 模块移除**：fragment 分组、列类型推断、DDL 评分流整链删除；内核分派谓词迁至中立位置，xfs / ext4 页检测共用，行为不变。

## v1.1.6 (2026-10-03)

backup、logminer、多库的全面体检与旧模式 list 语义对齐。

### 修复

- **旧模式（config database=X）`list table <pattern>` 的 schema 过滤全失效**：远古语义把参数当表名子串匹配（`list table public` 只出 pg_publication 系）。改与多库模式同一实现（引号感知的 `<schema>[.<table>]` 切分加 schema OID 过滤）。
- **logminer 跨段 WAL 续页越界**：截断段上用越界零填充"成功组装"出伪造记录，大长度布局下映射区越界。补段尾界。
- **wal_replayer 三处坏记录向量**：TDE 跳过缺长度检查（无符号下溢导致检查全失守）、FPW hole 零校验（写爆栈页、负长巨拷）、image_offset 未对记录长校验（读指到组装缓冲外）。
- **backup list 三命令路径漂移**：读侧用当前目录相对路径、写侧用字典根目录，换目录运行即 "Cannot open"；统一用字典根目录。
- **XFS 四处变长数组的块大小界**：超级块损坏可给出任意 block_size 撑爆栈，统一到 512~65536 的合法区间。
- **extract 认领混淆**：TOAST 表结构恰为（oid, int4, bytea），`(int,int,text)` 类 DDL 的校验对 chunk 页全数通过，chunk 数据会混入业务表输出。all 模式认领现排除 toast 类型页（chunk 收集路径独立，TOAST 重组不受影响）；分类器侧补严格判据链防业务表反向误判。

## v1.1.5 (2026-10-03)

hotfix：旧模式 `list table` 语义对齐。

### 修复

- **旧模式（config database=X）`list table <pattern>` 的 schema 过滤全失效**：远古语义把参数当表名子串匹配——`list table public` 只出 pg_publication 系。改走与多库模式同一实现（引号感知的 `<schema>[.<table>]` 切分、schema.dul OID 过滤、schema 内表名模糊匹配）。

## v1.1.4 (2026-10-02)

坏元数据与坏页的崩溃向量修复、诊断纠偏，以及 imp 闭环的转义缺陷。

### 修复

- **导出值未按 COPY text 转义（imp 闭环错列或丢行）**：值含分隔符 `|`、换行、反斜杠时原样写出行文件，`\copy` 导入必错列或整表失败。现按 PostgreSQL COPY text 规范转义（`\|`、`\n`、`\r`、`\\`），NULL 的 `\N` 原样直通。
- **imp 静默截断两处**：导入条目 4096 上限与 schema 256 上限触发时无声丢表或丢 DDL，现补明确告警。
- **recover_toast 诊断段：坏长度字无界拷贝导致栈溢出、尺寸虚胖**：chunk 长度记的是含 varlena 头的总长，而拷贝从头后起——坏页的 4 字节长度字（可达约 2^30）直接异常终止；正常路径 resolved_size 每 chunk 虚报。改为净载荷语义，补元组边界与来源页界两侧校验。同类问题两处一并修正。
- **TOAST chunk 载荷未按 TOAST_MAX_CHUNK_SIZE 封顶（栈写溢出）**：读取函数只按元组长（≤8K）封顶，坏页或误分类页里大于 2K 的伪 chunk 会溢出约 2K 的栈缓冲。真 chunk 恒 ≤2K（内核切片保证），封顶后恢复链无回归。
- **内联 datum 无上界越页读**：resolve 内联分支坏 4 字节头可解出巨量长度，按一页封顶（内核约束），属纵深防御。
- **行指针边界两处**：`get_vb_page_range` 缺 lp_len 下界（坏行指针头字段读越页）；`try_parse_tuple_with_ddl` 缺 `lp_off + lp_len ≤ BLCKSZ` 上界。

## v1.1.3 (2026-10-02)

修复一个自 v1.1.1 起的丢数据缺陷，以及两个坏元数据导致的崩溃向量。

### 修复

- **TOAST 扫描的 4 字节对齐过滤漏掉真指针（丢数据，v1.1.1 引入）**：v1.1.1 起四处 TOAST 指针字节扫描加了 `off % 4 == 0` 前置过滤，依据是"varlena 字段必然 intalign"——该假设只对**内联 4 字节头**的 varlena 成立。PostgreSQL 10~17 与 VastBase 的外部指针写入分支都是按当前位置裸拷贝、不做对齐。真实布局中 `(bool,text)` 的外部指针在 data 偏移 1、`(int2,text)` 在偏移 2——v1.1.1 / v1.1.2 在这两类常见表结构上会静默漏检全部 TOAST 指针。已删除四处对齐过滤，误检由 `vbdul_va_is_external_ondisk()` 的结构不变量继续承担。**建议 v1.1.1 / v1.1.2 用户全部升级。**
- **bmap btree 收集器三处加界（坏元数据崩溃，v1.1.1 引入）**：纯内部节点环（坏指针互指）导致无限递归栈溢出，加深度截断；叶子 numrecs 只封 1000 未按 block_size 封顶（4K 块仅容约 252 条）导致越界读；变长缓冲按超级块 block_size 无界分配。三处均已加界。

## v1.1.2 (2026-10-02)

热修：v1.1.1 引入的 BTREE fork 崩溃路径，与 TOAST 误检修复当时漏改的两处扫描点。

### 修复

- **`scan filesystem` 崩溃（BTREE fork 空指针解引用）**：v1.1.1 补的 bmap btree fork 真实遍历在 BTREE 分支无条件解引用上下文，而兼容入口恒传空——损坏或已删的 BTREE 格式目录 inode 走目录扫描的 fallback 路径时直接崩溃。现无上下文时优雅返回空 extents，并补上已分配缓冲的释放。
- **TOAST 指针裸扫误检（v1.1.1 漏数的另两处）**：`extract inode time ddl` 与 `extract free page ext ddl` 内还有两处弱校验循环，小端 int4 值叠加后续字节可满足旧校验，注册假 TOAST 指针导致后续提取找错文件。统一改走共享的结构不变量判定。
- `scan filesystem` 活集行标签由 `active` 改为 `allocated`（该集合即 inobt 权威遍历的已分配 inode）。

## v1.1.1 (2026-10-01)

修复 XFS 恢复过程中暴露的一批缺陷。

### 修复

- **TOAST 指针裸扫误检吞行**：表导出与恢复路径用裸字节扫描识别外部 TOAST 指针，无字段边界感知——行首 int4 字段值形如 0x??0112 的行会被误分流甚至彻底丢失。判定收敛为共享的结构不变量检查（对齐、rawsize / extsize / valueid 的语义界），两处扫描点统一。
- **XFS `scan filesystem`：DROP 后元数据未落盘导致漏扫**：XFS unlink 不释放 extents（挂在 AGI unlinked 链上推迟到 inode 回收），`sync()` 不触发回收，磁盘的 bnobt / AGI / inode fork 长期处于旧态。现在 `unload dict` 在写 XFS 元数据快照之前自动 sync、drop_caches、sync（需 root，非 root 给告警）。
- **XFS 并发扫描互删临时文件**：两个 vbdul 同时扫同一字典目录时，一方收尾删除共享临时文件，另一方随即读不到并退化为内存合并。临时文件名加进程号隔离。
- **`xfs_free_ext.dul` 打印语义**：原先打印的是页计数却标 "rows"，现同时打印真实行数（合并组数）与页数两个字段。
- **`unload dict` 全库被排除时假成功**：pg_database 有库但全部被排除名单或模板跳过时，此前打印 "built for 0 database(s)" 并返回 0；现明确报错（点名库数与排除名单）并以非 0 退出。
- **BTREE fork 解析**：XFS inode 的 bmap btree fork 此前返回空（大表 inode 全部丢失），补上真实 btree 遍历（指针按 maxrecs 偏移，复用目录扫描的收集器）。

## v1.1.0 (2026-09-30)

多库支持：一个实例几十个库一次处理，`unload dict` 建全部库字典，涉及库内对象的命令加 `db <name>` 子句；同时并入 db_version v6（VastBase G100 2.2.5）版本档与自定义表空间定位，并修复 v1.0.1 的 `imp` 必崩问题。

### 新增 — 多库支持

- **字典多库布局**：`config.dul` 的 `database` 留空即进入多库模式，`unload dict` 一次循环构建全部库的字典——库级字典按 `dict/<dboid>/*.dul` 分库存放，顶层只放共享与实例级字典（`db.dul`、`disk.dul`、`ext4_*.dul` 等），文件格式不变。`database` 有值时完全保持 v1.0.1 单库行为。单库失败可见化（obj.dul 哨兵机制），不再假报完成。
- **`db <name>` 子句**（多库模式）覆盖全部库内对象命令：`desc`、`list schema`、`list table`、`unload table`、`unload schema`、`verify table`、`dump filenodeid`、`imp`、`restore`、`recover table`、`logminer`。子句在分词前统一剥离，按命令白名单门控，引号感知。
- **`list db`**（`list database` 别名）：列出实例内全部数据库。
- **`unload db <spec> [meta|data]`**：批量导出整库或多库。spec 支持 `all`（减去排除名单，执行前列清单并要求确认）、`a,b,c` 显式列表（不受名单影响）、`<spec> except x,y`。`meta` 只导 DDL，`data` 只导数据。输出为 expdp 风格（库头、schema 缩进、每表 DDL 与导出行、收尾统计）。
- **输出文件 `<db>_` 前缀**：多库模式下导出文件统一带库名前缀，避免互相覆盖；单库模式保持 v1.0.1 原文件名。
- **`db_all_exclude` 配置**：`unload dict`、`unload db all`、`list schema|table db` 跳过的库名单，显式点名不受影响。
- **`unload dict using db.dul` 引导路径**：`global/pg_filenode.map` 定位不到 pg_database 时，手写 db.dul 当种子再解析各库；与 `config.database` 无关，两种字典布局都能跑。

### 新增 — 版本与表空间

- **db_version v6 = VastBase G100 2.2.5 版本档**：独立 catalog 表集（44 表）与 type_map（无 oradate，date 回 4 字节；name 列 64 字节）。未知 db_version 从静默回落 v3 改为明确报错退出。
- **自定义表空间定位**：表不在 pg_default 时按 `base/<dboid>/<filenode>` → `pg_tblspc/<tsoid>` 软链接多路径解析，定位不到时清单式报出尝试过的路径。heap 数据文件缺失从静默成功（0 行）改为明确失败，批量导出继续其余表并在收尾行计数。

### 修复

- **`imp` 一执行就崩溃（v1.0.1 起存在）**：三个约 9.9MB 的栈数组远超 8MB 线程栈，`imp` 任何形态必崩。改为堆分配并检查内存不足。
- **`unload dict` 全库被排除时假成功**：改为明确报错（点名库数与排除名单）并以非 0 退出。
- **logminer XID 恢复输出误吞 WAL 段名**："首段全数字"的判据把 24 位十六进制 WAL 段名也当 XID，原始输出被静默跳过；补十进制位数上界（≤10 位）。
- **pg_logminer 跨库 WAL 混入**：过滤用 first_db_oid（多库模式下恒 0）导致其他库的记录混进输出，改为按当前选中库过滤。
- **显示与导入路径旧账**：四个 display 函数分配未检查空指针、定长缓冲无界拷贝、`desc_backup` 的 strncpy 不补终止符，全部加固。
- **pg_partition 解析绕过版本分派**：回调硬编码 v3 列下标，其它版本档会读错列；改为版本分派加按列名取字段。"vastbase 且非 v3 被误分类为 PG 内核"一并修正。
- **`run()` 双 fclose 与导出 .sql 无写错检查**：补空指针复位与写错检查。
- **命令失败映射进程退出码**：EOF 返回最后一条真实命令的状态（exit / quit 不抹状态），未知命令与被 license 拦截为非 0，`license show` 恒 0，便于脚本化。

### 变更

- **`unload dict` 输出调整**：template0 / template1 静默跳过建字典（db.dul 仍列出）；每库开头一行 `unload db <名> (oid <oid>) dict:`；文件系统级字典行数块加头。
- **utl_file_dir 表全局排除**（VastBase utl_file 兼容包自建表）：unload schema 数据循环、DDL 枚举、unload db 清单计数三个口径一致排除；显式 `unload table` 点名仍可导出。
- **`unload db` 清单只计表数**，不再带数据量预估。

### 兼容性

- 单库模式（`config.dul` 的 `database` 有值）行为与 v1.0.1 一致。
- license 校验逻辑与 v1.0.1 一致：版本升级不作废 license，仍为无条件校验。

## v1.0.1 (2026-09-28)

### 修复

- **License 绕过漏洞（Release 版，自 v1.0.0 起存在）**：license 校验被套在"配置已加载"的条件里，于是只要没有 config.dul（删掉它，或把二进制换到别的目录运行），整段校验就被跳过——Release 版无需任何 license 即可使用全部功能。现改为无条件校验：config.dul 所在目录只用于定位 license.dul，找不到 config 时回落到可执行文件所在目录，再回落当前目录。失败仍进入受限模式（license request 逃生通道保留）。
- **`unload dict` 报 `Invalid num_mappings: 63 (max: 62)`**：pg_filenode.map 解析把老格式上限写死成 62，合法 map 被拒。实际各内核上限不同：PG ≤ 15 与 VastBase 老格式为 62（512 字节）、VastBase G100 2.2.5 为 63（520 字节）、PG 16+ 为 64（524 字节）、VastBase 4KB 块为 510（4096 字节）。现老格式统一按 64 放行，报错信息补充文件大小与格式。该错误在 `unload dict` 中不中断（只告警并退回 filenode=relid 兜底），但会导致整份 global map 被丢弃、系统表 filenode 全错——遇到此错修复后必须重跑。
- **字典文件空字段解析**（filesystem、disk、vg、pv、lv 共 7 处读，新增统一拆分函数）。
- **XFS B+tree 内部节点指针偏移**：指针起始位置按 maxrecs 计算、指针个数用 numrecs，原先按 numrecs+1 读，导致 `Invalid btree magic` 且静默返回 0 条；含三处递归共用缓冲与 is_v5 误判一并修复。
- **aarch64 包携带工具链指纹**：宿主 strip 无法处理交叉编译的 aarch64 ELF，而该行以 `|| true` 结尾，失败被静默吞掉，arm64 包一直带着编译器指纹发出（x86 因宿主同架构一直正常）。现按架构选用交叉工具链 strip / objcopy；UPX 无法压缩删过节区的 aarch64 ELF，改为清空 .comment 段内容而保留节头。
- **打包流程加硬门**：UPX 缺失、UPX 打包失败、工具链指纹残留都直接报错退出，不再静默产出未压缩或带指纹的包。

### 变更

- **Beta 与发布版标志分离**：版本串不再硬编码 Beta 前缀（旧版发布版会显示 "Beta 1.0.0 (Release)"，两个标志互相矛盾）。现版本串只含版本号，由构建宏在 banner 与 `--version` 里追加标志：Beta 构建显示 `1.0.1 (BETA)`，发布构建显示 `1.0.1 (Release)`；版本串自动拼接，避免手工漂移。

## v1.0.0 (2026-06-26)

首个正式版本。

### 核心恢复引擎

- 字典自动构建（`unload dict`）：直接从数据文件解析系统表，无需数据库在线
- Heap 页面精确解析：Tuple 字段解码、MVCC 事务可见性判断
- 大字段支持：TOAST 内联 PGLZ 压缩与外部 chunk 存储，1MB 以上完整恢复
- 复杂类型：JSONB（JB 二进制格式）、数组（int[] / text[] / bool[] / numeric[]）、number（ora_number 兼容）
- 分区表：pg_partition 字典解析，`unload table ... partition <name>` 按分区导出
- 无字典模式：`unload table <filenode_id> column <types>` 仅凭 OID 与类型导出
- 离线校验：`verify table` 数据一致性检查
- 导入配套：`imp` 自动生成 psql 导入脚本

### 数据库支持

- VastBase 全版本：v0（PG11 内核）、v1（PG14）、v2（PG16）、v3（VastBase 6.0）、v5（PG17）
- PostgreSQL 11 ~ 17（`db_type postgresql`）
- `db_type` 与 `db_version` 双参数分派独立解析路径

### 文件系统级恢复（数据库无法启动 / 文件被删）

- **ext4**：inode 扫描、extent tree 遍历、空闲块扫描、删除时间过滤提取
- **XFS**：AGFL 环形队列扫描、BNO / CNT B+tree 遍历、RMAP 反向映射、已删除目录项解析、inode 扫描、空闲碎片扫描
- **LVM**：VG / PV / LV 自动识别，多设备聚合扫描
- **rm -rf 数据目录**：`scan fs datadir` 按目录树定位，`extract fs datadir` 提取到 restore/
- 多级 B+tree 遍历、页面指纹分组、in-use bitmap 排除

### DROP TABLE / TRUNCATE 恢复（无备份）

- `scan filesystem free [parallel N]`：空闲空间扫描定位被释放的数据页面（ext4 空闲块 / XFS 空闲碎片）
- `list free page ext col <n> [tlen <m>]`：按字段数或 tuple 长度过滤缩小范围
- `extract free page using vb_ddl`：DDL 匹配提取并自动关联 TOAST
- `unload table <name> recover`：一步导出为可导入 SQL

### ProBackup 集成

- `list backup` / `list archivelog` / `show backup <id>`：backup.control 解析、WAL 区间合并
- `unload dict backup [BACKUP_ID]`：从备份集构建字典
- `restore table`：从备份还原单表数据文件
- `recover table ... stop_at lsn|xid`：选择性恢复到指定 LSN / XID

### Logminer（WAL 误操作回滚）

- `logminer <start> <end>`：挖掘 WAL 中的 DML（VastBase / PostgreSQL 自适应）
- 过滤：按表、XID、LSN 区间、提交时间
- `logminer xid <XID> recover`：按事务反向恢复——DELETE 生成数据文件、UPDATE 生成反向 UPDATE SQL
- 大事务缓冲恢复（`nobuf` 直写模式）

### 授权与工程化

- Ed25519 签名 license 授权体系：`license request` 生成机器指纹申请文件，离线签发
- 静态编译加 UPX 压缩，零运行时依赖；Linux x86_64 / aarch64
- 坏块容错：损坏页面、I/O 报错自动跳过
- 只读安全：所有设备与文件以只读方式访问，输出仅写入独立目录
