### 慕课网Java高并发秒杀([课程](http://www.imooc.com/learn/587))-代码完整优化版

### 修改说明：
#### 1.修改数据库连接池 为 Druid
#### 2.请修改redis.properties 配置为你自己redis 的配置
#### 3.已对视频中一些错误和SQL进行优化，代码编写更优雅，更易懂

#### SQL脚本
```sql
CREATE DATABASE seckill;
USE seckill;

-- todo:mysql　Ver　5.7.12for Linux(x86_64)中一个表只能有一个TIMESTAMP
CREATE TABLE seckill(
`seckill_id` BIGINT NOT NUll AUTO_INCREMENT COMMENT '商品库存ID',
`name` VARCHAR(120) NOT NULL COMMENT '商品名称',
`number` int NOT NULL COMMENT '库存数量',
`start_time` TIMESTAMP  NOT NULL COMMENT '秒杀开始时间',
`end_time`   DATETIME   NOT NULL COMMENT '秒杀结束时间',
`create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
PRIMARY KEY (seckill_id),
key idx_start_time(start_time),
key idx_end_time(end_time),
key idx_create_time(create_time)
)ENGINE=INNODB AUTO_INCREMENT=1000 DEFAULT CHARSET=utf8 COMMENT='秒杀库存表';

-- 初始化数据
INSERT into seckill(name,number,start_time,end_time)
VALUES
('1000元秒杀iphone6',100,'2016-01-01 00:00:00','2016-01-02 00:00:00'),
('800元秒杀ipad',200,'2016-01-01 00:00:00','2016-01-02 00:00:00'),
('6600元秒杀mac book pro',300,'2016-01-01 00:00:00','2016-01-02 00:00:00'),
('7000元秒杀iMac',400,'2016-01-01 00:00:00','2016-01-02 00:00:00')

-- 秒杀成功明细表
-- 用户登录认证相关信息(简化为手机号)
CREATE TABLE success_killed(
`seckill_id` BIGINT NOT NULL COMMENT '秒杀商品ID',
`user_phone` BIGINT NOT NULL COMMENT '用户手机号',
`state` TINYINT NOT NULL DEFAULT -1 COMMENT '状态标识:-1:无效 0:成功 1:已付款 2:已发货',
`create_time` TIMESTAMP NOT NULL COMMENT '创建时间',
PRIMARY KEY(seckill_id,user_phone),/*联合主键*/
KEY idx_create_time(create_time)
)ENGINE=INNODB DEFAULT CHARSET=utf8 COMMENT='秒杀成功明细表'

SHOW CREATE TABLE seckill\G;#显示表的创建信息
```
***
```sql
-- 存储过程
DELIMITER $$ -- console ; 转换为$$
-- 定义存储过程
-- 参数：in：输入参数
-- out：输出参数
-- ROW_COUNT();返回上一条修改类型的SQL(INSERT, DELETE, UPDATE)的影响行数
-- row_count：0未修改数据，>0表示修改的行数，<0表示SQL错误或者未执行修改SQL
CREATE PROCEDURE seckill.execute_seckill
  (IN v_seckill_id BIGINT, IN v_phone BIGINT,
   IN v_kill_time  TIMESTAMP, OUT r_result INT)
  BEGIN
    DECLARE insert_count INT DEFAULT 0;
    START TRANSACTION;
    INSERT IGNORE INTO success_killed (seckill_id, user_phone, state,create_time) VALUES (v_seckill_id, v_phone,0,v_kill_time);
    SELECT ROW_COUNT() INTO insert_count;
    IF (insert_count = 0) THEN
      ROLLBACK;
      SET r_result = -1;
    ELSEIF (insert_count < 0) THEN
      ROLLBACK;
      SET r_result = -2;
    ELSE
      UPDATE seckill SET number = number - 1 WHERE seckill_id = v_seckill_id AND end_time > v_kill_time AND start_time < v_kill_time AND number > 0;
      SELECT ROW_COUNT() INTO insert_count;
      IF (insert_count = 0) THEN
        ROLLBACK;
        SET r_result = 0;
      ELSEIF (insert_count < 0) THEN
        ROLLBACK;
        SET r_result = -2;
      ELSE
        COMMIT;
        SET r_result = 1;
      END IF;
    END IF;
  END;
$$
-- 存储过程定义结束

DELIMITER ;
SET @r_result = -3;
CALL execute_seckill(1003, 11111111111, now(), @r_result);

--获取结果
SELECT @r_result;
```
***
