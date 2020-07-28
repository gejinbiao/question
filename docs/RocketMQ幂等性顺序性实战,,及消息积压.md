# RocketMQ幂等性顺序性实战, 及消息积压解决方案

# 2. 消息幂等性

生产者发送消息之后，为了确保消费者消费成功 我们通常会采用手动签收方式确认消费，MQ就是使用了消息超时、重传、确认机制来保证消息必达。

场景：
　1. 订单服务(生产者),点击结算订单之后需要付款,这时就会发送一条“结算”的消息到mq的broker中。
　2. 此时支付服务(消费者)监听到这条消息之后就会处理结算扣款的逻辑，然后手动签收订单告诉mq我已经消费完成了。
　3. 如果在结算的过程中程序出现了异常，我们就返回mq一个消费失败的状态，此时mq就会重新发送这条消息；
　　或者是由于网络波动支付服务一直没有响应消息的消费状态，mq也照样会重新发送这条消息。
　4. 那么这种情况下，支付服务(消费者)就会重复收到这条消息，如果不做任何判断就有可能会重复消费出现多次扣款的情况。
解决方案：
　　在发送消息的时候,我们可以生成一个唯一ID标识每一条消息，将消息处理成功和去重日志通过事物的形式写入去重表或缓存中。每次消费之前都先查一遍，如果存在就说明消费过了直接返回消费成功。

# 3. 消息顺序性

　消息有序是指按照消息的发送顺序来消费。例如我们有这样一个需求：我们通过读取sql的日志文件得到它的所有sql，然后通过发送消息给其他服务去执行这些sql来同步数据。这个时候顺序性就显得尤为重要了。比如我们先修改这条数据，然后删除数据。倘若消费的时候先删除了再修改这时候得到的数据就不一致了。

![NOXqG8.png](https://s1.ax1x.com/2020/07/03/NOXqG8.png)

解决方案：
　　我们往一个topic里面发送消息时，它默认会维护几个队列，是随机发送到这些队列里面的。消费者集群消费时，实际上是一个服务监听一个队列。我们发送消息时，对于同个数据的操作指定发送到一个队列就好了，这时消费者就是按照顺序来消费的。

# 4. 顺序幂等案例实战

　比如我们现在我们有下面9条消息要发送，分别是对用户、订单、商品的操作，最终的结果应该是“用户被删除”，“订单已完成”，“商品被下架”。如果就条消息随机指定队列发送的话，就可能用户先被删除了，然后进行修改；也有可能支付订单的过程比较慢一直没反馈，从而收到多条消息而重复支付。

生产者对同一数据的操作发送到同一队列

	public List<Data> getOrderList(){
        List<Data> list = new ArrayList<Data>();
        list.add(new Data(1,"注册用户"));
        list.add(new Data(2,"创建订单"));
        list.add(new Data(1,"修改用户"));
        list.add(new Data(2,"支付订单"));
        list.add(new Data(1,"删除用户"));
        list.add(new Data(3,"商品上架"));
        list.add(new Data(2,"完成订单"));
        list.add(new Data(3,"商品打折"));
        list.add(new Data(3,"商品下架"));
        return list;
    }
    
    /**
     * @Title:顺序发送
     * @author:吴磊  
     * @date:2019年5月4日 下午10:28:31
     */
    @RequestMapping("/send")
    public Object send() throws Exception{
        // 1. 获取操作数据
        List<Data> list = getOrderList();
        
        Message message = null;
        for(int i=0; i<list.size(); i++) {
            Data data = list.get(i);
            // keys是唯一标识, 用作重试幂等判断.
            String keys = data.getId()+"";
            // 2. 装载消息 (topic, 标签, key, 消息体)
            message = new Message(PayOrderlyProducer.TOPIC,"test_tag",keys,data.getType().getBytes());
            /* 3. 投递消息
             *       每一个topic默认为4个queue, 我们可以指订队列发送.
             *   send(Message msg, MessageQueueSelector selector, Object arg),会将arg作为队列下标传
             *   给MessageQueueSelector中MessageQueue的arg,从而选出具体的queue.
             */
            SendResult sendResult =  payOrderlyProducer.getProducer().send(message, new MessageQueueSelector() {
                @Override
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                    int id = (int) arg;
                    int index = id % mqs.size();//对queue总数取模
                    return mqs.get(index);
                }
            },data.getId());
         System.out.printf("发送结果=%s, sendResult=%s ,orderid=%s, type=%s\n", sendResult.getSendStatus(), sendResult.toString(),data.getId(),data.getType());
        }
        return "OK";
    }

消费者手动签收消息，如果消费失败就重复消费，如果重试次数过多就通知运营人员手动处理。

	public PayOrderlyConsumer() throws MQClientException {

        consumer = new DefaultMQPushConsumer(consumerGroup);
        // 指定mq服务器地址
        consumer.setNamesrvAddr(NAME_SERVER);
        // 指定消费策略：这里是从最后一条消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);

        //默认是集群方式，可以更改为广播，但是广播方式不支持重试
        //consumer.setMessageModel(MessageModel.CLUSTERING);
        consumer.subscribe(TOPIC, "test_tag");

        /**
         * MessageListenerOrderly是单线程的消费
         */
        consumer.registerMessageListener( new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                MessageExt msg = msgs.get(0);
                int times = msg.getReconsumeTimes();
                try {
                    System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), new String(msgs.get(0).getBody()));
                    //模拟异常
                    String str = new String(msgs.get(0).getBody());
                    if(str.contains("订单")) {
                        int i=1/0;
                    }
                    
                    //做业务逻辑操作
                    System.out.println("消费成功");
                    return ConsumeOrderlyStatus.SUCCESS;
    
                } catch (Exception e) {
                    System.out.println("重试次数"+times);
                    //如果重试2次不成功，则记录，人工介入
                    if(times >= 2){
                        System.out.println("重试次数大于2，记录数据库，发短信通知开发人员或者运营人员");
                        return ConsumeOrderlyStatus.SUCCESS;
                    }
                    e.printStackTrace();
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
            }
        });
        consumer.start();
        System.out.println("consumer start ...");
    }

执行结果：可以看到id相同的数据(对同一数据的操作),都发送到同一队列了，然后消费者对每一个队列进行顺序消费，消费失败就会触发重试机制。

![NOjEM4.png](https://s1.ax1x.com/2020/07/03/NOjEM4.png)

# 5. 消息积压解决方案　

如果生产环境中consumer半夜挂掉了，项目数据量大的情况下第二天可能有一千万条消息积压在broker中。过多的数据积压不仅占用磁盘空间，还会影响MQ性能。那么这个时候应该怎么办呢。
　　修复consumer，然后慢慢消费吗？比如rocketmq默认一个topic是4个queue，假如处理一条消息需要1秒,用4个consumer同时进行消费,也需要10000000÷4÷60=42分钟才能消费完。那新的消息怎么办呢？
解决方案：
　　由于堆积的topic里面message queue数量固定，即使我们这时增加consumer的数量，它也不能分配到message queue。这时我们可以写一个分发程序做一个临时topic队列扩充，来提高消费者能力。程序从旧的topic中读取到新的topic,只是新的topic的queue可以指定多一点(理论上可以无限扩充，正常1000以内)，然后启动更多的consumer在临时新的topic里面消费。
![NOjuIx.png](https://s1.ax1x.com/2020/07/03/NOjuIx.png)

　