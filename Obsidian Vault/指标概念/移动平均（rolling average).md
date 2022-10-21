例子一： 股票价格移动平均
```SQL
SELECT
   date_time,
   stock_price,
   TRUNC(AVG(stock_price)
         OVER(ORDER BY date_time ROWS BETWEEN 3 PRECEDING AND CURRENT ROW), 4)
         AS moving_average
FROM stock_values;
```
移动平均定义：
$$ \mathrm{rolling\_average}=(\mathrm{stock\_price}_{row}+\mathrm{stock\_price}_{previous\_row}+\mathrm{stock\_price}_{row-2}+\mathrm{stock\_price}_{row-3})/4 $$
