# spring boot + mybatis-plus + gradle使用

## 依赖

```Gradle
plugins {
    id 'org.springframework.boot' version '2.6.13'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
    id 'java'
}
implementation 'org.postgresql:postgresql'
implementation 'com.baomidou:mybatis-plus-boot-starter:3.5.3.1'
```

## 连接配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://120.26.242.221:5432/postgres
    username: postgres
    password: password
```

## service 层一样。。。

## dao 层一样。。。

## 实体类

```Java
@TableName(value ="test_sc.user_password")
--mysql 是直接表名就行
--pgsql 需要加上schema 名称 

```

| schema_name | tb_name |
| -------------- | --------- |
| test_sc | user_password |

## 其他类型

未完待续。。。。