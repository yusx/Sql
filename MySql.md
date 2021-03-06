
MySql常用SQL
====

## 从一个表中取一个字段更新另一个表的字段 
```sql
update a inner join b on a.bid=b.id set a.x=b.x,a.y=b.y;
```

## group by 后取最小值 
Solution1:
```sql
SELECT 
    t1.* 
FROM 
    your_table t1
JOIN (
    SELECT MIN(value) AS min_value, dealer
    FROM your_table 
    GROUP BY dealer
) AS t2 ON t1.dealer = t2.dealer AND t1.value = t2.min_value
```
Solution2:
```sql
SELECT 
    t1.* 
FROM 
    your_table t1
LEFT JOIN your_table t2
ON t1.dealer = t2.dealer AND t1.value > t2.value
WHERE t2.value IS NULL
```

## 查询树形菜单

1. 建表
```sql
CREATE TABLE `sys_menu` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `parent_id` bigint(20) DEFAULT NULL,
    `menu_name` varchar(50) DEFAULT NULL,
    `url` varchar(200) DEFAULT '',
    `perms` varchar(200) NOT NULL DEFAULT '' ,
    `type` int(11) DEFAULT NULL ,
    `icon` varchar(50) DEFAULT '' COMMENT,
    `order_num` int(11) DEFAULT NULL,
    `remark` varchar(100) DEFAULT '',
    `update_time` timestamp NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `perms` (`perms`),
    KEY `parent_id` (`parent_id`),
    KEY `order_num` (`order_num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
 
2. 创建存储过程
```sql
CREATE FUNCTION `getChildLst`(rootId INT)
	RETURNS varchar(1000)
BEGIN
	DECLARE sTemp VARCHAR(1000);
	DECLARE sTempChd VARCHAR(1000);

	SET sTemp = '$';
	SET sTempChd =cast(rootId as CHAR);

	WHILE sTempChd is not null DO
	SET sTemp = concat(sTemp,',',sTempChd);
	SELECT group_concat(id) INTO sTempChd FROM sys_menu where FIND_IN_SET(parent_id,sTempChd)>0;
	END WHILE;
	RETURN sTemp;
END
```

3. 传入pid查出子id
```sql
SELECT getChildLst(1);
    
SELECT * FROM sys_menu 
WHERE FIND_IN_SET(id, getChildLst(1));
```

## 根据生日计算年龄
```sql
SELECT
	id,
	DATE_FORMAT(birthday,"%Y-%m-%d") birthday,
	CURDATE() ,
	(YEAR(now())-YEAR(birthday)-1) + ( DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') ) as age
FROM
	t_user 
```
说明：DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d'),这里是生日的日期和当前日期做一个比较，如果生日日期小于等于当前日期，说明已过生日，就+1岁，所以计算的是周岁


## 统计前7天的平均值

1. 建表
```sql
CREATE TABLE `test` (
  `kid` bigint(20) NOT NULL AUTO_INCREMENT ，
  `outid` varchar(50) DEFAULT NULL ，
  `ioflag` varchar(2) DEFAULT NULL ，
  `OpDT` datetime DEFAULT NULL COMMENT ，
  PRIMARY KEY (`kid`),
  KEY `outid` (`outid`),
  KEY `indate` (`OpDT`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ；
```

2.查询语句
```sql
SELECT a.outid,a.OpDT,a.num,
    ROUND((SELECT SUM(his.num) 
FROM (SELECT SUBSTR(OpDT,1,10) as OpDT,COUNT(*) as num
          FROM test
          WHERE outid = '123' 
          GROUP BY SUBSTR(OpDT,1,10)) his
WHERE his.OpDT >= DATE_ADD(a.OpDT,INTERVAL -7 DAY) and his.OpDT < a.OpDT)/7,1) as 7num 
FROM
(SELECT
	outid,SUBSTR(OpDT,1,10) as OpDT,ioflag,COUNT(*) as num
FROM test b
WHERE outid = '123' and SUBSTR(OpDT,1,10) BETWEEN '2017-03-07' and '2017-03-14'
GROUP BY SUBSTR(OpDT,1,10)) a
```

## 动态增量跟新
当增量数据大于100W的时候只更新100W，否则全部更新

```sql
UPDATE operate_mark o SET last_id = (
    CASE 
		WHEN (SELECT (MAX(a.id) - o.start_id) as num FROM tableName a) > 1000000 THEN  (o.start_id+1000000)
		WHEN (SELECT (MAX(a.id) - o.start_id) as num FROM tableName a) <= 1000000 THEN (SELECT max(id) FROM tableName) 
    END
)
WHERE o.table_name='tableName'
```

## 删除重复数据 
```sql
DELETE FROM `table` WHERE id IN (
    SELECT all_duplicates.id FROM (
        SELECT id FROM `table` WHERE (`title`, `SID`) IN (
            SELECT `title`, `SID` FROM `table` GROUP BY `title`, `SID` having count(*) > 1
        )
  ) AS all_duplicates 
LEFT JOIN 
    (SELECT id FROM `table` GROUP BY `title`, `SID` having count(*) > 1) AS grouped_duplicates 
    ON all_duplicates.id = grouped_duplicates.id 
WHERE grouped_duplicates.id IS NULL
)	
```

## MySql修改存储过程定义者
```sql
update mysql.proc set DEFINER='root@localhost' WHERE DEFINER='mysql@%' AND db='your_db_name';
```

## LEFTJOIN 和 case when 结合使用
```sql
LEFT JOIN challengesRead 
ON challenges.userID = CASE 
WHEN challenges.userID = $var THEN challengesRead.userID 
WHEN challenges.opponentID = $var THEN challenges.opponen END;


SELECT
	wo.wokey,
	wo.woname,
	p.posname,
	mo.posid,
	wo.posid AS woposid,
	wo.jobtype
FROM
	workorder wo
LEFT JOIN maintenanceobject mo ON wo.moid = mo.id
LEFT JOIN position p ON p.id = CASE
WHEN wo.jobtype = 3 THEN
	wo.posid
ELSE
	mo.posid
END;

```

## update 和 case when 结合使用

```
UPDATE table
SET table.date  = 
CASE
WHEN (
	id = 1
	AND table.date < '2019-01-01'
) THEN
	'2019-02-04' 

WHEN (id = 2
AND table.date <= '2019-01-01'
) THEN
	'2019-02-04'
ELSE
	planstartdate
END
WHERE
	id in (1,2);
```	

## MYSQL查询数据库中列

```
SELECT
	*
FROM
	information_schema. COLUMNS
WHERE
	table_schema = 'your_db_name'
AND COLUMN_NAME = 'column_name'
ORDER BY
	table_name,
	ordinal_position;
```

## mysql定时任务

1.查看当前是否已开启事件调度器，可执行如下SQL：

```
　SHOW VARIABLES LIKE 'event_scheduler';
```
2.开启事件调度器：

　SET GLOBAL event_scheduler = 1;
　也可以在配置my.cnf文件 中加上 event_scheduler = 1
　或
　SET GLOBAL event_scheduler = ON;
 
 为保证重启后自动启用事件：
 在配置文件中的[mysqld]部分添加 event_scheduler=ON
 
3.创建事件语法：
```
CREATE EVENT [IF NOT EXISTS] event_name
ON SCHEDULE schedule -- 定义执行的时间和时间间隔
[ON COMPLETION [NOT] PRESERVE]  -- 定义事件是一次执行还是永久执行，默认为一次执行，即NOT PRESERVE
[ENABLE | DISABLE] -- 定义事件创建以后是开启还是关闭，以及在从上关闭。如果是从服务器自动同步主上的创建事件的语句的话，会自动加上DISABLE ON SLAVE
[COMMENT 'comment']
DO sql_statement;
```
4.简单实例
 1）创建需要定时执行的存储过程
 ```
 DROP PROCEDURE IF EXISTS `test`;
 DELIMITER ;;
 CREATE DEFINER=`root`@`localhost` PROCEDURE `test`()
 BEGIN
 INSERT INTO table_a (create_date) VALUES (NOW())
 END
 ;;
 DELIMITER ;
 ```
2）创建事件

```
CREATE EVENT IF NOT EXISTS `test`
ON SCHEDULE EVERY 1 MINUTE 
ON COMPLETION PRESERVE
DO CALL test();
```

