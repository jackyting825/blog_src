---
title: springboot 接收参数统一除去参数值前后的空串
date: 2021-07-27 11:01:58
tags: 
  - java
  - spring boot
---

>springboot 接收参数统一除去参数值前后的空串

```java

  @ControllerAdvice(basePackages = {"com.bz"})
  public class StringNoSpaceAdvice {

      /**
      * 去除get方式的参数空格
      *
      * @param binder
      */
      @InitBinder
      public void initBinder(WebDataBinder binder) {
          // 创建 String trim 编辑器
          // 构造方法中 boolean 参数含义为如果是空白字符串,是否转换为null
          // 即如果为true,那么 " " 会被转换为 null,否者为 ""
          StringTrimmerEditor propertyEditor = new StringTrimmerEditor(false);
          // 为 String 类对象注册编辑器
          binder.registerCustomEditor(String.class, propertyEditor);
      }

      /**
      * 去除post/put requestBody实体中的空格
      *
      * @return
      */
      @Bean
      public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
          return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder
                  .deserializerByType(String.class, new StdScalarDeserializer<Object>(String.class) {
                      @Override
                      public String deserialize(JsonParser jsonParser, DeserializationContext ctx)
                              throws IOException {
                          return StringUtils.trimWhitespace(jsonParser.getValueAsString());
                      }
                  });
      }

  }


```