@Slf4j
public class ResponseMessageProcess extends ProcessFunction<ResponseMessage, IndexMessage> {

    private transient GeneralizedMessage message = new GeneralizedMessage();
	

在flink中, processFunction应该是从JM序列化传输到TM的, 
因为上面这个message在调用open的时候就会消失. 不加transient, 就会报错没有实现序列化接口.