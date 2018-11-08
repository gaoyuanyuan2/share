# Spring Boot REST
## 基本概念	
 REST = RESTful = Representational State Transfer，is one way of providing interoperability between computer systems on the Internet.
 
 [参考资源](https://en.wikipedia.org/wiki/Representational_state_transfer)    

## 1、目标
<br>1.  理解“资源操作”（Manipulation of resources through representations）
<br>POST、GET、PUT、DELETE
<br>幂等方法
<br><br>2.  理解“自描述消息” （Self-descriptive messages）
<br>Content-Type
<br>MIME-Type
<br><br>3.  扩展“自描述消息”
<br>HttpMessageConvertor
## 2、理解幂等
<br>1.  PUT 幂等
<br>初始状态：0
<br>修改状态：1 * N
<br>最终状态：1
<br><br>2.  DELETE 幂等
<br>初始状态：1
<br>修改状态：0 * N
<br>最终状态：0
<br><br>3.  POST 非幂等
<br>初始状态：1
<br>修改状态：1 + 1 =2
<br>N次修改： 1+ N = N+1
<br>最终状态：N+1
<br><br>幂等/非幂等 依赖于服务端实现，这种方式是一种契约

## 3、自描述消息
<br>1.  请求头

`Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8`

<br>第一优先顺序：`text/html` -> `application/xhtml+xml` -> `application/xml`
<br>第二优先顺序：`image/webp` -> `image/apng`
<br><br>2.  自描述消息处理器
<br>所有的 HTTP 自描述消息处理器均在 messageConverters（类型：`HttpMessageConverter`)，这个集合会传递到 RequestMappingHandlerAdapter，最终控制写出。
<br>messageConverters，其中包含很多自描述消息类型的处理，比如 JSON、XML、TEXT等等
<br>以 application/json 为例，Spring Boot 中默认使用 Jackson2 序列化方式，其中媒体类型：application/json，它的处理类 MappingJackson2HttpMessageConverter，提供两类方法：
<br><br>1) 读read* ：通过 HTTP 请求内容转化成对应的 Bean
<br><br>2) 写write*： 通过 Bean 序列化成对应文本内容作为响应内容

## 4、扩展自描述消息
<br>1.  实现 AbstractHttpMessageConverter 抽象类
<br><br>1)  supports 方法：是否支持当前POJO类型
<br><br>2)  readInternal 方法：读取 HTTP 请求中的内容，并且转化成相应的POJO对象（通过 Properties 内容转化成 JSON）
<br><br>3)  writeInternal 方法：将 POJO 的内容序列化成文本内容（Properties格式），最终输出到 HTTP 响应中（通过 JSON 内容转化成 Properties ）  
<br><br>2.  HttpMessageConverter 执行逻辑：
<br>读操作：尝试是否能读取，canRead 方法去尝试，如果返回 true 下一步执行 read
<br>写操作：尝试是否能写入，canWrite 方法去尝试，如果返回 true 下一步执行 write
<br><br>3. 编码
<br><br>1.  Properties 格式（application/properties+person)
<br>需要扩展
```properties
person.id = 1
person.name = yan
```
<br><br>2. json -> properties
<br>请求地址：`/person/json/to/properties`
<br>请求头
```properties
Accept=application/properties+person
Content-Type=application/json
```
<br>请求体
```json
{
    "id":7,
    "name":"yan"
}
```
<br>响应体
```properties
#Written by web server
#Tue Sep 18 15:35:00 CST 2018
person.name=yan
person.id=7
```
<br><br>3.properties -> json
<br>请求地址：`/person/properties/to/json`
<br>请求头
```properties
Content-Type=application/properties+person
Accept=application/json
```
<br>请求体
```properties
person.name=yan
person.id=7
```
<br>响应体
```json
{
    "id":7,
    "name":"yan"
}
```
<br><br>3. java 代码
<br><br>1)  配置
```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {

        converters.add(new PropertiesPersonHttpMessageConverter());
    }

}
```
<br><br>2)  controller
```java
    @PostMapping(value = "/person/json/to/properties", produces = "application/properties+person")
    public Person personJsonToProperties(@RequestBody Person person) {
        // @RequestBody 的内容是 JSON
        // 响应的内容是 Properties
        return person;
    }

    @PostMapping(value = "/person/properties/to/json", consumes = "application/properties+person", produces = 
            MediaType.APPLICATION_JSON_UTF8_VALUE)
    public Person personPropertiesToJson(@RequestBody Person person) {
        // @RequestBody 的内容是 Properties
        // 响应的内容是 JSON
        return person;
    }
```
<br><br>3) 自定义HttpMessageConverter
```java
public class PropertiesPersonHttpMessageConverter extends AbstractHttpMessageConverter<Person> {

    public PropertiesPersonHttpMessageConverter() {
        super(MediaType.valueOf("application/properties+person"));
        setDefaultCharset(Charset.forName("UTF-8"));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(Person.class);
    }
    /**
     * 讲请求内容中 Properties 内容转化成 Person 对象
     * @throws IOException
     * @throws HttpMessageNotReadableException
     */
    @Override
    protected Person readInternal(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, 
            HttpMessageNotReadableException {

        /**
         * person.id = 1
         * person.name = yan
         */
        InputStream inputStream = inputMessage.getBody();
        Properties properties = new Properties();
        // 将请求中的内容转化成Properties
        properties.load(new InputStreamReader(inputStream, getDefaultCharset()));
        // 将properties 内容转化到 Person 对象字段中
        Person person = new Person();
        person.setId(Long.valueOf(properties.getProperty("person.id")));
        person.setName(properties.getProperty("person.name"));
        return person;
    }

    @Override
    protected void writeInternal(Person person, HttpOutputMessage outputMessage) throws IOException, 
            HttpMessageNotWritableException {
        OutputStream outputStream = outputMessage.getBody();
        Properties properties = new Properties();
        properties.setProperty("person.id", String.valueOf(person.getId()));
        properties.setProperty("person.name", person.getName());
        properties.store(new OutputStreamWriter(outputStream, getDefaultCharset()), "Written by web server");
    }
}
```
## 5、 注意

`@RequestMappng` 中的 consumes 对应 请求头 `Content-Type`

`@RequestMappng` 中的 produces 对应 请求头 `Accept`

## 6、思考
<br>1.  当 Accept 请求头未被制定时，为什么还是 JSON 来处理？
<br>查看源码：
<br>DelegatingWebMvcConfiguration的父类WebMvcConfigurationSupport->addDefaultHttpMessageConverters方法，可以看出messageConverters插入顺序。
从插入顺序可以看出Spring Boot 中默认使用 Jackson2 序列化方式，采用轮训的方式去逐一尝试是否可以 HttpMessageConverter<T>->canWrite(POJO) ,如果返回 true，说明可以序列化该
 POJO 对象。
<br><br>2.  优先级是默认的是可以修改吗？
<br>是可以调整的，通过extendMessageConverters 方法调整
<br>同一个WebMvcConfigurerAdapter中的configureMessageConverters方法先于extendMessageConverters方法执行
<br>可以理解为是三种方式中最后执行的一种，不过这里可以通过add指定顺序来调整优先级，也可以使用remove/clear来删除converter
<br>使用converters.add(xxx)会放在最低优先级（List的尾部）
<br>使用converters.add(0,xxx)会放在最高优先级（List的头部）



