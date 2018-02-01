MySql常用SQL
====

##从一个表中取一个字段更新另一个表的字段 
```sql
update a inner join b on a.bid=b.id set a.x=b.x,a.y=b.y;
```
##group by 后取最小值 

    Solution1:
    
    SELECT t1.* FROM your_table t1
    JOIN (
      SELECT MIN(value) AS min_value, dealer
      FROM your_table 
      GROUP BY dealer
    ) AS t2 ON t1.dealer = t2.dealer AND t1.value = t2.min_value
    
    Solution2:
    
    SELECT t1.* FROM your_table t1
    LEFT JOIN your_table t2
    ON t1.dealer = t2.dealer AND t1.value > t2.value
    WHERE t2.value IS NULL

[参考][1]

## mysql 查询树形菜单 ##

 1. 建表

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

 

 2. 创建存储过程

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


3. 传入pid查出子id

    select getChildLst(1);
    
    select * from sys_menu 
       where FIND_IN_SET(id, getChildLst(1));


## 根据生日计算年龄 ##

    select
    	id,
    	DATE_FORMAT(birthday,"%Y-%m-%d") birthday,
    	CURDATE() ,
    	(year(now())-year(birthday)-1) + ( DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') ) as age
    from
    	t_user 

说明：DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d')
      这个是生日的日期和当前日期做一个比较，
      如果生日日期小于等于当前日期，说明已过生日，就+1岁，计算的是周岁


## 统计前7天的平均值 ##

 1. 建表

    CREATE TABLE `access_record_inout_temp2` (
      `kid` bigint(20) NOT NULL AUTO_INCREMENT ，
      `outid` varchar(50) DEFAULT NULL ，
      `ioflag` varchar(2) DEFAULT NULL ，
      `OpDT` datetime DEFAULT NULL COMMENT ，
      `school_code` varchar(50) DEFAULT NULL ，
      `faculty_code` varchar(50) DEFAULT NULL ，
      `major_code` varchar(50) DEFAULT NULL ，
      `class_code` varchar(50) DEFAULT NULL ，
      `sex` varchar(10) DEFAULT NULL，
      PRIMARY KEY (`kid`),
      KEY `outid` (`outid`),
      KEY `indate` (`OpDT`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8 ；


    SELECT a.outid,a.OpDT,a.num,
    	ROUND((SELECT SUM(his.num) 
    	 FROM (SELECT SUBSTR(OpDT,1,10) as OpDT,COUNT(*) as num
    				FROM access_record_inout_temp2
    				WHERE outid = '168111563136' 
    				GROUP BY SUBSTR(OpDT,1,10)) his
       WHERE his.OpDT >= DATE_ADD(a.OpDT,INTERVAL -7 DAY) and his.OpDT < a.OpDT)/7,1) as 7num 
    FROM
    (SELECT
    	outid,SUBSTR(OpDT,1,10) as OpDT,ioflag,COUNT(*) as num
    FROM access_record_inout_temp2 b
    WHERE outid = '168111563136' and SUBSTR(OpDT,1,10) BETWEEN '2017-03-07' and '2017-03-14'
    GROUP BY SUBSTR(OpDT,1,10)) a
	
## 动态增量跟新 ##
	当增量数据大于100W的时候只更新100W，否则全部更新

	UPDATE own_operate_mark o SET last_id = (
		CASE 
			WHEN (SELECT (MAX(a.id) - o.start_id) as num from access_record a) > 1000000 THEN  (o.start_id+1000000)
		  WHEN (SELECT (MAX(a.id) - o.start_id) as num from access_record a) <= 1000000 THEN  (SELECT max(id) FROM access_record) END
	)
	where o.table_name='access_record_inout_temp2'

## mysql 删除重复数据 ## 

DELETE FROM `table` WHERE id IN (
  SELECT all_duplicates.id FROM (
    SELECT id FROM `table` WHERE (`title`, `SID`) IN (
      SELECT `title`, `SID` FROM `table` GROUP BY `title`, `SID` having count(*) > 1
    )
  ) AS all_duplicates 
  LEFT JOIN (
    SELECT id FROM `table` GROUP BY `title`, `SID` having count(*) > 1
  ) AS grouped_duplicates 
  ON all_duplicates.id = grouped_duplicates.id 
  WHERE grouped_duplicates.id IS NULL
)	
