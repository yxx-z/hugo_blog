# SpringBoot配置文件yml内容加密

在使用Springboot时，通常很多信息都是在application.yml中直接明文配置的，比如数据库链接信息，redis链接信息等等。但是这样是不安全的。
<!--more-->
***

## 前言
开发或者工作中，很多配置都是以明文的方式放在yml文件里，但是这样是不符合安全规范的。在公司的要求下，配置文件中敏感信息需要加密。

## 解决
### 引入pom依赖
```java
  <!-- jasypt -->
  <dependency>
      <groupId>com.github.ulisesbocchio</groupId>
      <artifactId>jasypt-spring-boot-starter</artifactId>
      <version>3.0.3</version>
  </dependency>
```
***

### yml文件中配置加密钥匙
```yml
  jasypt:
    encryptor:
      password: pwd
```
<p style="color:red">注：pwd为密钥，自己随意配置</p>
这里的配置是在开发时候，毕竟如果上线的话密钥放在yml里 加不加密一个样。</br>
线上启动的方式后面说

***

### 加密敏感数据
未加密前mysql部分配置
```yml
spring:
  #定义数据源
  datasource:
    url: jdbc:mysql://localhost:3306/test?useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=GMT%2B8&zeroDateTimeBehavior=convertToNull
```
开始加密</br>
这里在单元测试里运行加密
```java
    @Autowired
    private StringEncryptor encryptor;

    @Test
    void encrypt(){
        System.out.println("mysqlUrl: " + encryptor.encrypt("jdbc:mysql://localhost:3306/test?useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=GMT%2B8&zeroDateTimeBehavior=convertToNull"));
    }
```
运行单元测试结果
```text
mysqlUrl: ROeydT7oNHA3CCU+WPpgVc8wNdGS2flt/hlt3ot6YlXadrF4TP4P3ehdP+jMpnmW0XscNR2LYQzlIW9sQRsmEp62Mwk86afyLl3WiJr+aYijHuVIBeetc9uvGgCNcA5Jjr0stCfXgU5pRAbyaD+OK6Hz08iByAD0gq3PoOGo4H6yhSL3+HKo0a0bczgAJSIvxT+xr04chuIu/1QiFODNke+s6lGY+UEucAZikq0UUI4=
```

***

### 将加密后数据配置到yml文件中
规则：ENC(加密后的数据) </br>
例：
```yml
spring:
  #定义数据源
  datasource:
    url: ENC(ROeydT7oNHA3CCU+WPpgVc8wNdGS2flt/hlt3ot6YlXadrF4TP4P3ehdP+jMpnmW0XscNR2LYQzlIW9sQRsmEp62Mwk86afyLl3WiJr+aYijHuVIBeetc9uvGgCNcA5Jjr0stCfXgU5pRAbyaD+OK6Hz08iByAD0gq3PoOGo4H6yhSL3+HKo0a0bczgAJSIvxT+xr04chuIu/1QiFODNke+s6lGY+UEucAZikq0UUI4=)
```
启动项目，调用接口发现能够正常获取数据库中数据 表明加解密成功

### 线上使用
前面是将密钥配置在yml文件中，在项目上生产的时候这样肯定不行，毕竟把密钥放里面和没加密没有区别。</br>
现在将上面配置的密钥删除
```yml
  jasypt:
      encryptor:
        password: pwd
```
删除以上部分。但是加密过的数据，例如mysql的url ENC(....)部分保持不变。
然后在启动jar包的时候增加一下命令
<p style="color:red">-Djasypt.encryptor.password=pwd</p>

```java
java -jar -Djasypt.encryptor.password=pwd XXX-xxxx.jar
```

## 结束
完事，划水

---

> 作者: map[avatar:<nil> email:yangxx@88.com link:<nil> name:yxx]  
> URL: http://localhost:1313/configenc/  

