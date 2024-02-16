
# 结构图

[[tomcat/_resources/tomcat/9abc1eab57d6ed8ffb88bb7fadfd0898_MD5.jpeg|Open: 截屏2023-12-16 20.54.03.png]]
![[tomcat/_resources/tomcat/9abc1eab57d6ed8ffb88bb7fadfd0898_MD5.jpeg]]


## 启动类 org.apache.catalina.startup.Bootstrap


##  源码启动配置
[[tomcat/_resources/tomcat/d7c6bbcfa7e098cf3330432f65ec10a2_MD5.jpeg|Open: 截屏2023-12-16 20.37.06.png]]
![[tomcat/_resources/tomcat/d7c6bbcfa7e098cf3330432f65ec10a2_MD5.jpeg]]



## 启动时许图

![[tomcat/_resources/tomcat/2d360654ee46d97c04dfeed2c732632b_MD5.png]]



## web.xml 配置

```xml

  
<web-app>  
  <display-name>Archetype Created Web Application</display-name>  
  <context-param>    <!--    spring config   -->  
    <param-name>contextConfigLocation</param-name>  
    <param-value>/WEB-INF/applicationContext.xml</param-value>  
  </context-param>  
  <listener>    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
  </listener>  
  <servlet>    <!--    spring mvc DispatcherServlet-->  
    <servlet-name>appServlet</servlet-name>  
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
    <init-param>      <param-name>contextConfigLocation</param-name>  
      <param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>  
    </init-param>  
    <load-on-startup>1</load-on-startup>  
  </servlet>  
  <servlet-mapping>    <servlet-name>appServlet</servlet-name>  
    <url-pattern>/</url-pattern>  
  </servlet-mapping>  
</web-app>

```



###  启动时加载（ web.xml  ） 中 配置的 servlet   


```java

/**
org.apache.catalina.startup.ContextConfig#configureContext
*/
private void configureContext(WebXml webxml) {

	for (ServletDef servlet : webxml.getServlets().values()) {  
		//....
	}
}
```



![[servlet 创建初始化.svg]]