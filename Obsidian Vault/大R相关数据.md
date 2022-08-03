# 用户付费分层类型及数值范围
当前有动态（ 过去N天，比如N = 30 ) 分层和历史付费分层两种，具体分层数值
![[动态静态层级付费.png]]
[[历史分层的两种时间]]
## 1. 付费分层
利用PL/MySQL 计算用户的最后付费分层状态
**计算逻辑：** #计算逻辑
1. 最后登出（付费表）得出相应付费时间 $t$
2. 时间 $t$ 前的付费数据 
3. 计算付费次数、付费总额

### 1.1 预准备数据表格
最后付费表 A 数据列:
`last_paytime,last_paydate, pid, uuid, last_productid, last_recharge_amount,`
付费表 B 数据列[^1] ： 
`pid, uid, paytime, paydate, productid, recharge_amout`

[^1]:  实际的数据列需要根据数据记录的格式调整

最后登出（登入）表 C 数据列
`last_logdate, last_logtime, pid, uuid, last_diamond, last_gift_ticket, last_twistticket`

### 1.2 “历史“ 付费数据表 hist_pay
动态付费下付费用户最后付费状态代码：
```MySQL
SELECT A.pid, A.uid, last_paydate,  productid, recharge_amount, paydate  
FROM A
LEFT OUTER JOIN B 
ON A.pid = B.pid 
WHERE date_diff('day', paydate, last_paydate) between 0 and {{Variable: N}}
```
**附注：**
- 用户最后登入时付费状态，上述代码中 B -> C, paydate->logdate
- 考虑历史付费分层时， `WHERE` 语句可以略去




### 1.3. 聚合历史付费数据
通过步骤2得到的“历史”付费信息计算用户的付费次数、付费总额等聚合数据

### 1.3.1 PID维度不区分时段数据
按pid 分组计算
```MySQL
SELECT pid, sum(recharge_amout) "历史付费", 
		count(*) "付费次数", count(distinct paydate) "付费天数"
FROM hist_pay
GROUP BY pid
```

### 1.3.2 按PID和日期分组计算
```MySQL
SELECT pid, paydate, sum(recharge_amout) "当日付费", 
		count(*) "付费次数"
FROM hist_pay
GROUP BY pid,paydate
```
后续计算指标：每个用户单日最大付费金额

>> 只是考虑聚合数据情况的话，我们并不需要通过第一步、第二步的操作，只需要对历史付费数据表进行操作

```MySQL
SELECT pid,max(paydate) "最后付费日期", 
		   min(paydate) "首次付费日期", 
		   sum(rmb) "付费金额", 
		   count(paytime) "付费次数", 
		   count(paydate) "付费天数" 
FROM pay_table
GROUP BY pid
```
```
```
# 2. 付费大R产生监测

不同于付费状态监测，我们要以玩家的首次付费为起点追踪其后续的付费情况，当然也可以考虑以用户注册为起点来研究用户的破冰付费时长数据。


## 2.1 需要的数据表格

用户首次付费数据表 First_Pay:
`pid,uid, first_paytime,first_paydate, first_productid, first_recharge_amount`
用户付费数据表 Pay (开服至今):
`pid,uid,paytime, paydate, productid, recharge_amount`
用户注册 Reg (数数只有21年10月1号至今的注册数据)：
`pid,uid,regtime,regdate,channelid`

## 2.2 累计付费数据
首次付费N天内的付费数据表 After_Pay 
```MySQL
SELECT First_Pay.pid, First_Pay.uid, 
		first_paydate,  productid, recharge_amount, paydate  
FROM First_Pay
LEFT OUTER JOIN Pay 
ON First_Pay.pid = Pay.pid 
WHERE date_diff('day', first_paydate, paydate) <= {{Variable: N}}
```

## 2.3 大R产生监测
付费数据表的数据处理：各用户每日的付费数据 After_Daily_Pay
```MySQL
SELECT pid, paydate, sum(recharge_amount) pay_today, count(*) num_pay  
FROM After_Pay
GROUP BY pid, paydate
```
用户的累计付费情况
```MySQL
SELECT pid, paydate, pay_today, num_pay
       cume_dist() OVER (PARTITION BY pid
                    ORDER BY paydate ASC) AS acumu_pay
FROM After_Daily_Pay
ORDER BY pid, paydate
```

历史付费达到一定值：
玩家的历史付费表 T_h，当某个用户达到大R级别付费时进入该表，
需要有一个状态量表示其刚达成大R级别
① p1+△p >= 1400
② p1< 1400

