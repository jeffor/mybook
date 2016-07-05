# 餐厅评分计算

#### 1. 各维度分支因子及其计算方式（目前8个纬度）
- 餐厅有效订单量(valid_order_count_score)
 1. 计算一个周期内的 valid_order_count
  ```
  select count(id) from eleme_order where status_code = 2 group by restaurant_id
  ```
  2. 归一化计算当前周期的 valid_order_count_score
  ```
  该值需要归一映射，映射公式为 
  valid_order_score =100*arctan(valid_order_count*coordinate)*2/π, (coordinate=0.01)
  ```
  3. 基于该纬度历史分值迭代计
   
- 餐厅接单率
  1. 计算一个周期内接单率 (single_rate)
  ```   
  接单率 = 已接单数量 / 订单总量
  已接单数量
  select count(eleme_order.id) from order_process_record left join eleme_order on order_process_record.order_id = eleme_order.id where order_process_record.from_status = 0 and order_process_record.to_status = 2 group by eleme_order.restaurant_id
  订单总量
  select count(id) from eleme_order where status_code != -4 group by restaurant_id
  ```
  2. 基于该纬度历史分值迭代计算

- 接单前餐厅取消订单率

  1. 计算一个周期接单前取消订单率 (cancle_rate_before)
  ```
  接单前取消订单数量/订单总量
  接单前取消订单数量
  select count(eleme_order.id) from order_process_record left join eleme_order on order_process_record.order_id = eleme_order.id where order_process_record.from_status = 0 and order_process_record.to_status = -1 group by eleme_order.restaurant_id
  订单总量:
  select count(id) from eleme_order where status_code != -4 group by restaurant_id
  ```
  2. 基于该纬度历史分值迭代计算

- 接单后取消订单率
  1. 计算一个周期接单后取消订单率 (cancle_rate_after)
    ```
    接单后取消订单率=接单后取消订单数量/已接单数量
    接单后取消订单数量
    select count(eleme_order.id) from order_process_record left join eleme_order on order_process_record.order_id = eleme_order.id where order_process_record.from_status = 2 and order_process_record.to_status = -1 and order_process_record.process_group in (5,11,12,13,14,15) group by eleme_order.restaurant_id
    订单总量:
    select count(id) from eleme_order where status_code != -4 group by restaurant_id
    ```
  2. 基于该纬度历史分值迭代计算

- 用户投诉率
  1. 计算一个周期用户投诉率 (complaint_rate)
  ```
  	用户投诉率=用户投诉量/订单总量
	用户投诉量
		select count(id) from order_complaint group by restaurant_id
	订单总量
		select count(id) from eleme_order where status_code != -4 group by restaurant_id
  ```
  2. 基于该纬度历史分值迭代计算


- 订单退单率
  1. 计算一个周期退单率 (refund_rate)
  ```
	退单率=用户申请退单数/餐厅接单数量
	用户申请退单数
		select count(eleme_order.id) from order_refund_record left join eleme_order on order_refund_record.order_id = eleme_order.id where order_refund_record.from_status = 0 and order_refund_record.to_status = 2 group by eleme_order.restaurant_id
	订单总量
		select count(eleme_order.id) from order_process_record left join eleme_order on order_process_record.order_id = eleme_order.id where order_process_record.from_status = 0 and order_process_record.to_status = 2 group by eleme_order.restaurant_id
  ```
  2. 基于该纬度历史分值迭代计算

7. 订单仲裁率
  1. 计算一个周期仲裁率 (arbitration_rate)
  ```
	仲裁率=仲裁数量/申请退单数量
	仲裁数量
		select count(eleme_order.id) from order_refund_record left join eleme_order on order_refund_record.order_id = eleme_order.id where order_refund_record.from_status = 3 and order_refund_record.to_status = 4 group by eleme_order.restaurant_id
	申请退单数量
		select count(eleme_order.id) from order_refund_record left join eleme_order on order_refund_record.order_id = eleme_order.id where order_refund_record.from_status = 0 and order_refund_record.to_status = 2 group by eleme_order.restaurant_id
  ```
  2. 基于该纬度历史分值迭代计算

- 餐厅处理退单时长
  1. 计算一个周期餐厅退单总时长 (refund_cost)
  ```
	“用户申请退单的时间”至“餐厅给出处理结果的时间”
    
	select t1.created_at as begin_time, t2.created_at as end_time, (t2.created_at - t1.created_at) as interval_time from order_refund_record as t1 left join order_refund_record as t2 on t1.order_id = t2.order_id left join eleme_order on t2.order_id = eleme_order.id where t1.from_status = 0 and t1.to_status = 2 and t2.from_status = 2 and (t2.to_status = 3 or t2.to_status = 6)  
  ```
  2. 计算平均退单时长
  ```
    退单总时长/退单数目
    退单数量
		select count(eleme_order.id) from order_refund_record left join eleme_order on order_refund_record.order_id = eleme_order.id where order_refund_record.from_status = 0 and order_refund_record.to_status = 2 group by eleme_order.restaurant_id
  ```
  3. 基于该纬度历史分值迭代计算

- 餐厅接单时长
  1. 计算一个周期餐厅接单总时长 (order_cost)
  ```
	用户支付成功 至 餐厅接单
	select (order_process_record.created_at - eleme_order.active_at) from eleme_order left join order_process_record on eleme_order.id = order_process_record.order_id where order_process_record.from_status = 0 and order_process_record.to_status = 2
  ```
  2. 计算平均接单时长
  ```
     接单总时长/接单数目
     已接单数量
  select count(eleme_order.id) from order_process_record left join eleme_order on order_process_record.order_id = eleme_order.id where order_process_record.from_status = 0 and order_process_record.to_status = 2 group by eleme_order.restaurant_id
  ```
  3. 基于该纬度历史分值迭代计算

> 可能产生的调整变动说明:
>   1. 归一化公式中的 coordinate 参数
>   2. 迭代计算周期
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
   k: 当前分值迭代计算比重，目前定为0.1
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
