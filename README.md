# quantiance
基于tushare示例的一个进程+协程方式实现股票数据的采集

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2019-09-19 20:34:55
# @Author  : Cypress McCarthy Bai 
# @Link    : mr.baishu@gmail.com
# @Version : 1.0

import tushare
import tushare as ts
import numpy as np

import time
import datetime

import asyncio
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor   

import pymysql

ts.set_token('a137962cac5e3806501ec16e4007e4475ab7d8cd703204bb5a260d29')
pro = ts.pro_api('a137962cac5e3806501ec16e4007e4475ab7d8cd703204bb5a260d29')

dbconfig = {'host': '127.0.0.1',
            'user' : 'root',
            'passwd' : 'hao123',
            'db' : 'stock',
            'port' : 3306,
            'charset' : 'utf8'
            }

db = pymysql.connect(**dbconfig)
cursor = db.cursor()

stocksPool = ['603912.SH',
          '600183.SH',
          '300053.SZ',
          '300618.SZ',
          '002049.SZ',
          '002892.SZ',
          '300576.SZ',
          '300672.SZ']


startime = '20180901'
timeTemp = datetime.datetime.now() - datetime.timedelta(days = 1)
endtime = timeTemp.strftime('%Y%m%d')


# 用于清洗某个股票某天的股价 并存储数据库   
async def get_StockDayPrice(rawData):
    #清洗数据中的'nan',使用-1替代
    # db = pymysql.connect(**dbconfig)    
    # cursor = db.cursor()
    
    cleanData = list(map(lambda x: -1 if x =='nan' else x, rawData))   
    state_dt=(datetime.datetime.strptime(cleanData[1], "%Y%m%d")).strftime('%Y-%m-%d')
    #print(state_dt,cleanData)
    try:
        sql_insert = """INSERT INTO stock_all(
                        state_dt, stock_code,
                        open_dt, close_dt,
                        high, low, vol, amount, 
                        pre_close, amt_change,
                        pct_change) 
                        VALUES
                        ('%s', '%s','%.2f', 
                        '%.2f','%.2f','%.2f',
                        '%i','%.2f','%.2f',
                        '%.2f','%.2f')""" %\
                        (state_dt,str(cleanData[0]),\
                         float(cleanData[2]),float(cleanData[5]),\
                         float(cleanData[3]),float(cleanData[4]),\
                         float(cleanData[9]),float(cleanData[10]),\
                         float(cleanData[6]),float(cleanData[7]),\
                         float(cleanData[8]))
        cursor.execute(sql_insert)
        db.commit()
        await asyncio.sleep(0.1)
        print("hello word")
    except Exception as err:
        print("error",err)
    # cursor.close()
    # db.close()

#调用协程
def get_StocksDailyPrice(stock_name):
 
    try:
        df= pro.daily(ts_code = stock_name,
                     start_date = startime,
                     end_date = endtime)
        #print(df.head())
        a = df.values.tolist()  #a: 中间变量
    except Exception as aa:
        print('No datga code:' + stock_name)
        
    t1 = time.time()
    
    try:
        loop = asyncio.get_event_loop()
        tasks = [get_StockDayPrice(i) for i in a]
        loop.run_until_complete(asyncio.wait(tasks))
        loop.close()
    except Exception as err:
        print("here,here",err)

    t2 = time.time()
    duration = t2 - t1
    print("aaa",duration)


#调用进程，将不同的股票放到不同的协程池
def main(): 
    executor = ProcessPoolExecutor(max_workers = 3)
    t1 = time.time()
    try:
        executor.map(get_StocksDailyPrice, stocksPool)
    except Exception as err:
        print('cannot', err)
        
    t2 = time.time()
    duration = t2 - t1
    print("aaa",duration)
    executor.shutdown()


if __name__ == '__main__':
    main()
    print('All Finished!')
 
```
