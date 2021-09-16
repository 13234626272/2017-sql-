# 2017-sql-
2017阿里慢sql优化挑战赛

# 背景
2017年 阿里云数据库挑战赛！不服来战！
第一季：挑战玄惭之 慢SQL性能优化赛
玄惭，真名罗龙九，阿里云DBA专家，负责阿里云RDS线上稳定以及专家服务团队，人称“大师”！
他，经历阿里历年双11考验，积累了6年对阿里云数据库用户的运维、调优、诊断等丰富的经验。
他，就是本次挑战赛的出题人！


# 销售员表
CREATE TABLE `a` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `seller_id` bigint(20) DEFAULT NULL,
  `seller_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin DEFAULT NULL,
  `gmt_create` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=32744 DEFAULT CHARSET=utf8;

# 订单表
CREATE TABLE `b` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `seller_name` varchar(100) DEFAULT NULL,
  `user_id` varchar(50) DEFAULT NULL,
  `user_name` varchar(100) DEFAULT NULL,
  `sales` bigint(20) DEFAULT NULL,
  `gmt_create` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=32744 DEFAULT CHARSET=utf8;

# 销售员和订单关联表
CREATE TABLE `c` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(50) DEFAULT NULL,
  `order_id` varchar(100) DEFAULT NULL,
  `state` bigint(20) DEFAULT NULL,
  `gmt_create` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=458731 DEFAULT CHARSET=utf8;


## 待优化sql - 优化前执行时间为190ms
select a.seller_id,a.seller_name,b.user_name,c.state 
from a,b,c 
where a.seller_name=b.seller_name  
and b.user_id=c.user_id 
and c.user_id=17  
and a.gmt_create BETWEEN DATE_ADD(NOW(), INTERVAL - 600 MINUTE)
AND  DATE_ADD(NOW(), INTERVAL 600 MINUTE)
order  by  a.gmt_create;

EXPLAIN select a.seller_id,a.seller_name,b.user_name,c.state 
from a,b,c 
where a.seller_name=b.seller_name  
and b.user_id=c.user_id 
and c.user_id=17  
and a.gmt_create BETWEEN DATE_ADD(NOW(), INTERVAL - 600 MINUTE)
AND  DATE_ADD(NOW(), INTERVAL 600 MINUTE)
order  by  a.gmt_create;

# 优化思路
1. 首先，由于是多表联查，所以优先需要找到他的驱动表，因为我们知道，join关联查询时，是以数据量较的表作为驱动表的，所以先查询出那张表的数据量最小
SELECT COUNT(*) FROM a WHERE a.gmt_create BETWEEN DATE_ADD(NOW(), INTERVAL - 600 MINUTE) AND  DATE_ADD(NOW(), INTERVAL 600 MINUTE);  // 1

SELECT COUNT(*) FROM b;  // 10000

SELECT COUNT(*) FROM c WHERE c.user_id=17;  // 1

a表和c表都是只有一条数据， 但由于原sql中是有 order  by  a.gmt_create 所以从索引的利用率来讲，应将a表设为驱动表

## 开始考虑创建索引

# A表上创建索引：
Alter table a add index ind_a_gmt_create(gmt_create);
# 如果回表取数据量较大，可以考虑将关联字段和查询字段冗余到索引中
Alter table a add index ind_a_gmt_create(gmt_create,seller_name,seller_id);

# B表上创建索引：
Alter table b add index ind_b_seller_name(seller_name);
# 如果回表取数据量较大，可以考虑将关联字段和查询字段冗余到索引中
Alter table b add index ind_b_seller_name1(seller_name,user_name,user_id);

# C表创建索引：
Alter table c add index ind_c_user_id(user_id);
# 如果回表取数据量较大，可以考虑将关联字段和查询字段冗余到索引中
Alter table c add index ind_c_user_id(user_id,state);

## 添加完索引后， 此时执行sql，会发现 只有C表使用到了索引，而a，b表并没有使用到，这是为什么呢？

EXPLAIN EXTENDED select a.seller_id,a.seller_name,b.user_name,c.state 
from a,b,c 
where a.seller_name=b.seller_name  
and b.user_id=c.user_id 
and c.user_id=17  
and a.gmt_create BETWEEN DATE_ADD(NOW(), INTERVAL - 600 MINUTE)
AND  DATE_ADD(NOW(), INTERVAL 600 MINUTE)
order  by  a.gmt_create;

SHOW WARNINGS;

# 警告分析
a表：gmt_create使用了varchar来存储，在5.6支持更高时间精度后，将会发生隐式转换。

b表：a表和b表的seller_name字段在COLLATE定义上不一致，也会导致隐式转换。

c表：b表和c表的user_id字段都定义为了varchar，但是SQL传入为数字，也会导致隐式转换。


alter table  a modify column  gmt_create datetime;  
alter table  a modify column  seller_name varchar(100) ;
alter table  c modify column user_id  bigint;









