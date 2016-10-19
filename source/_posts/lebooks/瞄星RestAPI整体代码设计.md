---
title: 瞄星RestAPI整体代码设计
date: 2016-08-30 20:16:24
categories: 
- 程序设计
tags:
- lebooks-private
---

# 瞄星RestAPI整体代码设计

1. RestController
  1. Controller类添加@RestController注解（必须Spring4.0以上）
  2. 标注RequestMapping的方法必须返回org.springframework.http.ResponseEntity， 必须增加responseBody和http status code
  3. 可预料的异常与错误，必须返回通俗易懂的responsebody与http status code

2. 错误管理与分类
  1. 抽象类ErrorResponseBody定义通用的错误响应体
  
    ```java
      public abstract class ErrorResponseBody {
        private String error_code;
        private String error_msg;
      }
    ```
  2. 接口Error定义了错误的行为：必须能转换为
  
    ```java
      public interface Error {
          ErrorResponseBody toResponseBody();
      }
    ```
  3. 定义一组ErrorResponseBody子类区分不同类型的ErrorResponseBody
  4. 定义一组Error实现类（枚举）区分不同错误
    ```java
      public enum ServiceError implements Error {
          // 01: auth
          ValidCodeInconsistent(20101, "valid code inconsistent"),
          ValidCodeExpire(20102, "valid code expire"),
          ValidCodeTooOften(20103, "valid code send too often(less than 60s)"),
      
          // 02: author
          AuthorNotFound(20201, "Author Not Found"),
      
          // 03: work
          WorkNotFount(20301, "Work Not Found");
      
          private int errorCode;
          private String errorMsg;
      
          private ServiceError(int errorCode, String errorMsg){
              this.errorCode = errorCode;
              this.errorMsg = errorMsg;
          }
      
          @Override
          public ServiceErrorResponseBody toResponseBody(){
              return new ServiceErrorResponseBody(this.errorCode + "", this.errorMsg);
          }
      }
    ```
  5. 业务逻辑通过throw ServiceException(Error)的形式传递错误类型并在Controller中转换为ErrorResponseBody
3. 使用apidoc根据java方法注释生成api文档
  使用方法[参考](http://apidocjs.com/)
