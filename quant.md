如何开发出一套选股模型，能够替代人工，选出能够跑赢大盘的股票组合。

下图是自己对量化的总结，包括回测和实盘两部分。
![在这里插入图片描述](https://img-blog.csdnimg.cn/de777599f14a47e2803240e5074c1419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5waW5nemp1,size_16,color_FFFFFF,t_70#pic_center)


实盘的目标不是跑出多好的结果，而是尽可能和回测的结果保持一致，如果存在较大的偏差，则需要弄清楚原因，列举一些原因：
* 回测数据和实盘数据不一致
* 交易对股票价格产生了很大的冲击
* 人为因素，比如受情绪影响干预了交易
* 程序BUG
* 行情数据和历史数据误差较大

确定交易周期(日/周/月/季度)，周期结束后，需要对数据进行维护，验证交易结果，具体来说就是回测本周期的数据，核对实盘和回测的结果，计算误差，发现问题。

程序化交易是最好的选择，如果人工交易，则不要看盘，避免受情绪影响，出现误操作。

当模型开始交易后，不要频繁修改模型，要对回测的结果有信心。