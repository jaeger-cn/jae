# jaeger示例

```
func JaegerInit() {
	//jaeger
	cfg := jaegercfg.Configuration{
		Sampler: &jaegercfg.SamplerConfig{
			Type:  jaeger.SamplerTypeConst,
			Param: 1,
		},
		Reporter: &jaegercfg.ReporterConfig{
			LogSpans:           true,
			LocalAgentHostPort: "127.0.0.1:6831",//地址
		},
		//链路追踪的名字
		ServiceName: "jaeger_name",
	}
	//直接控制台输出信息
	tracer, closer, err := cfg.NewTracer(jaegercfg.Logger(jaeger.StdLogger))
	if err != nil {
		panic(err)
	}
	//设置全局的tracer
	opentracing.SetGlobalTracer(tracer)
	defer closer.Close()
	parentSpan = tracer.StartSpan("create_order")//span命名

  //spanA:=opentracing.StartSpan("funcAAA",opentracing.ChildOf(parentSpan.Context()))
	//time.Sleep(time.Millisecond * 500)
	//spanA.Finish()
	parentSpan.Finish()
}
```

# RocketMQ延时示例

```
// CreateOrder host端口号
func CreateOrder(host string) {
	//顶单
	// mq
	rq, _ := rocketmq.NewProducer(
		producer.WithNsResolver(primitive.NewPassthroughResolver([]string{host})),
		//消息发送失败后重试的次数
		producer.WithRetry(2),
	)
	err := rq.Start()
	defer func(p rocketmq.Producer) {
		err = p.Shutdown()
		if err != nil {
		}
	}(rq)
	marshal, _ := json.Marshal([]interface{}{} /*结构体*/)

	msg := primitive.NewMessage("order_topic", marshal)

	//设置消息的延迟时间
	msg.WithDelayTimeLevel(3)
	//1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
	res, _ := rq.SendSync(context.Background(), msg)
	fmt.Println(res)
}
```

# RocketMQ消费示例

```
func DelOrder() {
	c, _ := rocketmq.NewPushConsumer(
		consumer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})),
	)
	c.Start()

	var order *OrderInfo
	c.Subscribe("order_topic", consumer.MessageSelector{}, func(c context.Context,msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
		for _, m := range msgs {
			json.Unmarshal(m.Body, &order)
		}
		return consumer.ConsumeSuccess, nil
	})
	defer c.Shutdown()

	spanA := opentracing.StartSpan("del_order", opentracing.ChildOf(parentSpan.Context()))
	time.Sleep(time.Millisecond * 500)
	spanA.Finish()
}
```



