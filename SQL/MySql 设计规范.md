## 数据库命名规范
1. 数据库对象名称必须使用小写字母并用下划线分割
2. 禁止使用mysql保留关键字
3. 见名识意，不超过32个字符
4. 临时表必须以tmp为前缀，日期为后缀
5. 备份库、表必须以bak为前缀，日期为后缀
6. 所有存储相同数据的列名和列数据类型必须一致


## 数据库基本设计规范
1. 所有表必须使用Innodb存储引擎
   - mysql5.5使用myisam默认引擎
2. 数据库和表的字符集统一使用utf8
3. 索引表和字段都要添加注释
   - 从一开始就进行数据字典的维护
4. 控制单表数据量的大小，建议500W行以内
   - 实际限制取决于文件系统，64位系统可以无视了
5. 谨慎使用mysql分区表（物理多文件，逻辑一个表）
6. 尽量做到冷热数据分离，减少表的宽度
   - 减少磁盘IO，保证热数据的内存缓存命中率
   - 更有效的利用缓存，避免读入无用的冷数据
   - 经常一起使用的列放到一个表中
7. 禁止在表中建立预留字段
   - 预留字段很难做到见名识义
   - 预留字段无法确认存储的数据类型
   - 对预留字段类型的修改，会对表进行锁定
8. 禁止在数据库中存储图片、文件等二进制数据
9. 禁止在线上做数据库压力测试
10. 禁止从开发环境、测试环境直连生产环境数据库

## 数据库索引设计规范
1. 限制每张表中的索引数量，建议单张表索引不超过5个
   - 提高查询，降低新增和更新
   - 5.6可以使用多个单列索引联合查询，但不如联合索引
2. 每个Innodb表必须有一个主键
   - 不使用更新频繁的列作为主键、不使用多列主键
   - 不使用UUID、MD5、HASH、字符串作为主键
   - 建议使用自增ID值
3. 常见索引列建议
   - select、update、delete语句的where从句中的列
   - 包含在order by、group by、distinct中的列
   - 多表 join 的关联列
4. 如何选择索引列的顺序（从左到右）
   1. 区分度最高(唯一性，如主键)的列放在联合索引的最左侧
   2. 尽量把字段长度小的列放在联合索引的最左侧
   3. 使用最频繁的列放在联合索引的最左侧
5. 避免建立冗余索引和重复索引
   1. 重复如：primary key（key）、index（id）、unique index（id）
   2. 冗余如：index（a, b, c）、index（a，b）、 index（a）
6. 对于频繁的查询优先考虑使用覆盖索引
   1. 避免Inoodb表进行索引的二次查找
   2. 可以把随机IO变为顺序IO加快查询效率
7. 尽量避免使用外键约束

## 数据库字段设计规范
1. 优先选择符合存储需要的最小的数据类型
   - 将字符串转化为数字类型存储（INET_ATON：IP转整形，INER_NTOA：整形转IP）
   - 对于非负数据采用无符号整型进行存储
   - varchar（N）中的N：代表的是字符数（中文），而不是字节数
   - 过大的长度会消耗更多的内存
2. 避免使用text、blob数据类型
   - 如必须使用，使用单独的扩展表存放
   - text和blob类型只能使用前缀索引
3. 避免使用enum数据类型（枚举）
   - 修改enum值需要使用alter语句
   - enum类型的order by 操作效率低，需要额外操作
   - 禁止使用数值作为enum值
4. 尽可能把所有列定义为 not null
   - 索引null列需要额外的空间来保存，占用更多空间（空、非空）
   - 进行比较和计算要对 null 值特别处理（is null）
5. 使用timestamp或datetime类型存储时间
   - timestamp 1970-01-01 00:00:01 ～ 2038-01-19 03:14:07
   - timestamp占用4字节，和int相同，但可读性更高
6. 财务相关的金融类数据，必须使用decimal类型
   - decimal为精准浮点数、在计算时不会丢失精度
   - 占用空间由定义的宽度决定
   - 可用于存储比bigint更大的数

## 数据库SQL开发规范
1. 建议使用预编译语句进行数据库操作（PREPARE）
   - 只传参数，比传递sql语句更高效
   - 相同语句可以一次解析，多次使用，提高处理效率
   - 避免 sql 注入
2. 避免数据类型的隐式转换（where id = "123";）
   - 隐式转换会导致索引失效
3. 充分利用表上已经存在的索引
   - 避免使用双 % 的模糊查询条件，或前置 % 查询条件，如：%name
   - 一个sql只能利用到复合索引中的一列进行范围查询
   - 使用left join 或 not exists 来优化 not in 操作，not in 会使索引失效
4. 程序连接不同的数据库使用不同的账号，禁止跨库查询
   - 方便数据库迁移和分库分表
   - 降低业务耦合度
   - 避免由权限过大而产生的业务风险
5. 禁止使用select * 进行查询
   - 消耗更多的cpu 和io以及网络带宽资源
   - 无法利用覆盖索引
   - 可减少表结构变更带来的影响
6. 禁止使用不含有字段列表的 insert 语句
   - insert into values （）
7. 避免使用子查询，可以把子查询优化为 join 操作
   - 子查询结果集无法使用索引
   - 子查询会产生临时表，如果数据量大会严重影响效率
   - 消耗过多cpu和io资源
8. 避免使用 join 关联太多的表
   - 每 join 一个表会多占用一部分内存（join_buff_size） 
   - 会产生临时表操作，影响查询效率
   - mysql最多允许关联61个表，建议不超过5个
9. 减少同数据库的交互次数
   - 数据库更适合处理批量操作（返还100还是1000，效率差不多）
   - 合并多个相同的操作到一起，可以提高处理效率
10. 使用 in 代替 or
    - in 的值不超过500个
    - in的操作可以更有效的利用索引，or 很少
11. 禁止使用order by rand() 进行随机排序
    - 会把所有符合条件的数据装载到内存中进行排序，消耗大量cpu和io
    - 推荐在程序中获取一个随机值，然后从数据库获取数据
12. 禁止在 where 从句中对列进行函数转换和计算
    - 对列进行函数转换或计算会导致无法使用索引（where date(createtime) = '20160209'）
    - 优化为：where createtime >= '20160209' and createtime < '20160209', 可以使用到索引
13. 在明显不会有重复值，使用union all ，而不是 union
    - union会把所有数据放到临时表再进行去重操作
14. 拆分复杂的大sql 为多个小sql
    - mysql的一个sql只能使用一个cpu 进行计算，拆分后可以并行执行提高效率

## 数据库操作行为规范
1. 超过100W行的批量写操作，要分批多次操作
   - 大批量写操作，会造成严重的主从延迟
   - binlog日志为row格式时会产生大量的日志
   - 避免产生大事务操作
2. 对于大表使用 pt-online-schema-change修改表结构
   - 避免主从延迟
   - 避免对表字段进行修改时的锁表
3. 禁止为程序使用的账号赋予 super 权限
   - 当达到最大连接数量限制时，还允许一个有 super 权限的用户连接
   - super 只能留给 DBA处理问题的账号使用
4. 对于程序连接数据库账号，遵循权限最小原则
   - 程序使用数据库账号，只能在一个DB下使用，不准跨库
   - 程序使用的账号原则上不准有 drop 权限
