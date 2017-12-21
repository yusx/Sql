########## ��һ������ȡһ���ֶθ�����һ������ֶ� ##########
update a inner join b on a.bid=b.id set a.x=b.x,a.y=b.y;


########## group by ��ȡ��Сֵ  ##########

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





##########  mysql ��ѯ���β˵�  http://blog.csdn.net/acmain_chm/article/details/4142971  ##########

1.����
CREATE TABLE `sys_menu` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `parent_id` bigint(20) DEFAULT NULL COMMENT '���˵�ID��һ���˵�Ϊ0',
  `menu_name` varchar(50) DEFAULT NULL COMMENT '�˵�����',
  `url` varchar(200) DEFAULT '' COMMENT '�˵�URL',
  `perms` varchar(200) NOT NULL DEFAULT '' COMMENT '��Ȩ(����ö��ŷָ����磺user:list,user:create)',
  `type` int(11) DEFAULT NULL COMMENT '����   0��Ŀ¼   1���˵�   2����ť  3 : ͼ��',
  `icon` varchar(50) DEFAULT '' COMMENT '�˵�ͼ��',
  `order_num` int(11) DEFAULT NULL COMMENT '����',
  `remark` varchar(100) DEFAULT '' COMMENT '��ע',
  `update_time` timestamp NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `perms` (`perms`),
  KEY `parent_id` (`parent_id`),
  KEY `order_num` (`order_num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='�˵�����';



2.�����洢����
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


3. ����pid�����id
select getChildLst(1);

select * from sys_menu 
   where FIND_IN_SET(id, getChildLst(1));


########## �������ռ������� ###########  

select
	id,
	DATE_FORMAT(birthday,"%Y-%m-%d") birthday,
	CURDATE() ,
	(year(now())-year(birthday)-1) + ( DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') ) as age
from
	t_user 

˵����DATE_FORMAT(birthday, '%m%d') <= DATE_FORMAT(NOW(), '%m%d') ��������յ����ں͵�ǰ������һ���Ƚϣ�
�����������С�ڵ��ڵ�ǰ���ڣ�˵���ѹ����գ���+1�꣬�����������


################ ͳ��ǰ7���ƽ��ֵ #######################

����
CREATE TABLE `access_record_inout_temp2` (
  `kid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id����Ӧaccess_record���kid��������',
  `outid` varchar(50) DEFAULT NULL COMMENT 'ѧ��',
  `ioflag` varchar(2) DEFAULT NULL COMMENT '����״̬ 0:�� 1:��',
  `OpDT` datetime DEFAULT NULL COMMENT 'ˢ��ʱ��',
  `school_code` varchar(50) DEFAULT NULL COMMENT 'У��code����',
  `faculty_code` varchar(50) DEFAULT NULL COMMENT 'Ժϵcode�����Զ������',
  `major_code` varchar(50) DEFAULT NULL COMMENT 'רҵcode�����Զ������',
  `class_code` varchar(50) DEFAULT NULL COMMENT '�༶code�����༶ֻ��code,û������',
  `sex` varchar(10) DEFAULT NULL COMMENT '�Ա�',
  PRIMARY KEY (`kid`),
  KEY `outid` (`outid`),
  KEY `indate` (`OpDT`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='�������:ʱ����м��';


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