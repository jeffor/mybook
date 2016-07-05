# 用户评分计算

#### 1. 各维度分支因子及其计算方式（目前8个纬度）
- 有效订单量
 1. 计算一个周期内的 valid_order_count
  ```
  select count(id) from eleme_order where status_code = 2 group by restaurant_id
  ```
  2. 归一化计算当前周期的 valid_order_count_score
  ```
  该值需要归一映射，映射公式为 
  valid_order_score =100*arctan(valid_order_count*coordinate)*2/π, (coordinate=1)
  ```
  3. 基于该纬度历史分值迭代计
   
- 无效订单量
  1. 计算一个周期内的无效订单数目
  2. 归一化计算当前周期的 invalid_order_count_score
  ```
  该值需要归一映射，映射公式为 
  invalid_order_score =100*arctan(invalid_order_count*coordinate)*2/π, (coordinate=1)
  ```
  3. 基于该纬度历史分值迭代计2. 归一化计算当前周期的 valid_order_count_score
  
- 交易平均金额
  1. 计算一个周期内交易总金额 (order_cost)
  2. 计算交易平均金额
  ```
    周期内交易总金额/周期内订单数目
  ```
  3. 基于该纬度历史分值迭代计算

- 取消订单率

  1. 计算当前周期内取消订单数目
  2. 计算当前周期内取消订单率
  3. 迭代计算取消订单分值


> 可能产生的调整变动说明:
>   1. 归一化公式中的 coordinate 参数
>   2. 迭代计算周期，目前7天为一个周期
>   3. 纬度数目的增减

#### 2. 各个参数迭代计算方式
 - 迭代周期
 ```
   目前可定为30天
 ```
 - 计算公式
 ```
   score[now] = score[current]*k + score[history]*(1-k)
   
   score[now]: 当前需要计算的目标分值
   score[current]: 基于当前周期交易数据计算的分值
   score[history]: 历史分值
   k: 当前分值迭代计算比重，目前定为0.3
 ```
 > kennel产生的调整变动说明
 > 1. 迭代周期
 > 2. 迭代公式的k值

#### 3. 调整计算说明
  - 调整范围
    - 见上述计算说明
  - 调整频率
    - 由于模型还未稳定，调整会比较频繁，基本2天一次
  - 调整后结果响应时间需求
    - 希望当天能响应调整，利用夜间计算，第二天早晨可出结果数据
  - 上线时间
    - 7月20日
