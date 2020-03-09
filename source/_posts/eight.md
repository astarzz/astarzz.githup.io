---
title: "UTC日期格式化问题"
date: 2019-5-7 10:33:10
categories: 
    - "日期"
toc_number: false
tags:
	- date format
	- UTC
	- json serialize
---
## 世界时间

### UTC
协调世界时，又称世界统一时间、世界标准时间、国际协调时间。由于英文（CUT）和法文（TUC）的缩写不同，作为妥协，简称UTC。
UTC是一个标准，而不是一个时区。UTC是一个全球通用的时间标准。全球各地都同意将各自的时间进行同步协调 (coordinated)，这也是UTC名字的来源：Universal Coordinated Time。
中国大陆、中国香港、中国澳门、中国台湾、蒙古国、新加坡、马来西亚、菲律宾、西澳大利亚州的时间与UTC的时差均为+8，也就是UTC+8。
<!--more-->
### GMT
格林尼治标准时间（旧译格林威治平均时间或格林威治标准时间；英语：GreenwichMeanTime，GMT）是指位于英国伦敦郊区的皇家格林尼治天文台的标准时间，因为本初子午线被定义在通过那里的经线。
地球每天的自转是有些不规则的，而且正在缓慢减速。所以，格林尼治时间已经不再被作为标准时间使用。现在的标准时间──协调世界时（UTC）──由原子钟提供。大体上GMT=UTC，但是不如UTC精准。
### CST
CST(中国标准时间)是UTC+8时区的知名名称之一，比UTC（协调世界时)提前8个小时与UTC的时间偏差可写为+08:00。CST也是我们最常见到的日期表示格式,无论在java程序，还是在服务器时间中。

## 问题
由于部门项目目前都是国内项目，所以日常日期方式一直采用CST。但是最近有个项目，应上级集团要求，接口数据出参日期必须统一采用UTC格式表示。
这里就要说说它们之间的不同，最根本区别的是时间差，CST比UTC晚8个小时，意味着本地项目的时间要减掉8个小时才是UTC时间。还有UTC的日期表示格式，不能随意自定义，它要满足**yyyy-MM-dd'T'HH:mm:ss.SSS'Z'**
但是解决起来却发现了几个问题：
1. 入参格式和出参不一致
项目的改造只是要求出参日期格式，但是入参并未改变，继续使用**yyyyMMddHHmmss**格式，所以我们需要一个工具可以分别定义格式化方式。
2. DateTimeFormat无效
因为出入参对象都是json格式，所以我们使用的是springboot默认的序列化工具jackson。而jackson也为我们提供了2个有用的注解 **@DateTimeFormat**和 **@JsonFormat**。
DateTimeFormat可以定义日期反序列化格式，即接口的入参格式，如 **@DateTimeFormat(pattern = "yyyyMMddHHmmss")**。
JsonFormat可以定义日期序列化格式，即接口出参格式，如 **@JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'",timezone = "UTC")**
```java
@DateTimeFormat(pattern = "yyyyMMddHHmmss")
@JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'",timezone = "UTC")
private Date birthday;
```
这样看似完美解决问题，但是使用中发现，DateTimeFormat并未生效。查找资料后来发现，@DateTimeFormat只会在类似@RequestParam的请求参数（url拼接的参数）上生效，如果@DateTimeFormat放到@RequestBody下是无效的。

## 解决方案
- JsonDeserializer
既然只是入参格式无法定义，而且是注解无效，那么我们就要考虑是否可以重写或者自定义转换方式，毕竟有spring强大的依赖注入。
几经周折，找到了 **JsonDeserializer**，它是jackson的反序列化工具，通过继承它实现自定义反序列化规则。
```java
/**
 * 自定义Jackson反序列化日期类型时应用的类型转换器,一般用于@RequestBody接受参数时使用
 */
public class DateJacksonConverter extends JsonDeserializer<Date> {
    private static String[] pattern = new String[]{"yyyyMMddHHmmss"};//这里可以定义多种格式

    @Override
    public Date deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {

        Date targetDate = null;
        String originDate = p.getText();
        if (StringUtils.isNotEmpty(originDate)) {
            try {
                targetDate = DateUtils.parseDate(originDate, DateJacksonConverter.pattern);
            } catch (ParseException pe) {
                throw new IOException(String.format(
                        "'%s' can not convert to type 'java.util.Date',just support timestamp(type of long) and following date format(%s)",
                        originDate,
                        StringUtils.join(pattern, ",")));
            }
        }


        return targetDate;
    }

    @Override
    public Class<?> handledType() {
        return Date.class;//这里定义需要处理的类型
    }
}
```

- 最后将我们自定义的反序列化工具注入Jackson2ObjectMapperFactoryBean
```java
@Configuration
public class ConverterConfig {
    @Bean
    public DateJacksonConverter dateJacksonConverter() {
        return new DateJacksonConverter();
    }

    @Bean
    public Jackson2ObjectMapperFactoryBean jackson2ObjectMapperFactoryBean(
            @Autowired
                    DateJacksonConverter dateJacksonConverter) {
        Jackson2ObjectMapperFactoryBean jackson2ObjectMapperFactoryBean = new Jackson2ObjectMapperFactoryBean();

        jackson2ObjectMapperFactoryBean.setDeserializers(dateJacksonConverter);
        return jackson2ObjectMapperFactoryBean;
    }

    @Bean
    public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter(
            @Autowired
                    ObjectMapper objectMapper) {
        MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter =
                new MappingJackson2HttpMessageConverter();
        mappingJackson2HttpMessageConverter.setObjectMapper(objectMapper);
        return mappingJackson2HttpMessageConverter;
    }
}
```