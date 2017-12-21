########## 从一个表中取一个字段更新另一个表的字段 ##########
update a inner join b on a.bid=b.id set a.x=b.x,a.y=b.y;


########## group by 后取最小值  ##########

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

https://dev.mysql.com/doc/refman/5.6/en/example-maximum-column-group-row.html





##########  mysql 查询树形菜单  http://blog.csdn.net/acmain_chm/article/details/4142971  ##########

1.建表
CREATE TABLE `sys_menu` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `parent_id` bigint(20) DEFAULT NULL COMMENT '父菜单ID，一级菜单为0',
  `menu_name` varchar(50) DEFAULT NULL COMMENT '菜单名称',
  `url` varchar(200) DEFAULT '' COMMENT '菜单URL',
  `perms` varchar(200) NOT NULL DEFAULT '' COMMENT '授权(多个用逗号分隔，如：user:list,user:create)',
  `type` int(11) DEFAULT NULL COMMENT '类型   0：目录   1：菜单   2：按钮  3 : 图表',
  `icon` varchar(50) DEFAULT '' COMMENT '菜单图标',
  `order_num` int(11) DEFAULT NULL COMMENT '排序',
  `remark` varchar(100) DEFAULT '' COMMENT '备注',
  `update_time` timestamp NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `perms` (`perms`),
  KEY `parent_id` (`parent_id`),
  KEY `order_num` (`order_num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='菜单管理';



2.创建存储过程
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


########## 根据生日计算年龄 ###########  

select
	id,
	DATE_FORMAT(birthday,"%Y-%m-%d") birthday,
	CURDATE() ,
	(year(now())-year(birthday)-1) + ( DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') ) as age
from
	t_user 

说明：DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') 这个是生日的日期和当前日期做一个比较，
如果生日日期小于等于当前日期，说明已过生日，就+1岁，计算的是周岁


################ 统计前7天的平均值 #######################

建表：
CREATE TABLE `access_record_inout_temp2` (
  `kid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id：对应access_record表的kid自增主键',
  `outid` varchar(50) DEFAULT NULL COMMENT '学号',
  `ioflag` varchar(2) DEFAULT NULL COMMENT '进出状态 0:进 1:出',
  `OpDT` datetime DEFAULT NULL COMMENT '刷卡时间',
  `school_code` varchar(50) DEFAULT NULL COMMENT '校区code：：',
  `faculty_code` varchar(50) DEFAULT NULL COMMENT '院系code：：自定义编码',
  `major_code` varchar(50) DEFAULT NULL COMMENT '专业code：：自定义编码',
  `class_code` varchar(50) DEFAULT NULL COMMENT '班级code：：班级只有code,没有名称',
  `sex` varchar(10) DEFAULT NULL COMMENT '性别',
  PRIMARY KEY (`kid`),
  KEY `outid` (`outid`),
  KEY `indate` (`OpDT`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='宿舍出入:时间表（中间表）';


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