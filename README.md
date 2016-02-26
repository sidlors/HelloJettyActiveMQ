# HelloJettyActiveMQ

Jetty con ActiveMQ


I wanted to figure out how to use Apache ActiveMQ in a web application that is running in Jetty, Also i wanted to use the Maven Jetty Plugin so i built this sample application which contains a , when i make GET request to servelt it takes value of message query parameter and publishes it as a TextMessage to a Queue, you can download the source code for the sample application from here First thing that i did is create a pom.xml file that looks like this

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
  http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.webspherenotes.jms</groupId>
  <artifactId>HelloJettyActiveMQ</artifactId>
  <version>1.0</version>
  <packaging>war</packaging>
  <name>HelloJettyActiveMQ</name>
  <description>Sample app to demonstrate how to use 
  ActiveMQ in Jetty</description>
  <dependencies>
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
      <version>2.4</version>
    </dependency>
    <dependency>
      <groupId>org.apache.activemq</groupId>
      <artifactId>activemq-core</artifactId>
      <version>5.5.0</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>1.5.11</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-jms</artifactId>
      <version>3.0.3.RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.apache.xbean</groupId>
      <artifactId>xbean-spring</artifactId>
      <version>3.9</version>
    </dependency>
  </dependencies>
  <build>
    <finalName>HelloJettyActiveMQ</finalName>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>1.5</source>
          <target>1.5</target>
        </configuration>
      </plugin>
   
      <plugin>
        <groupId>org.mortbay.jetty</groupId>
        <artifactId>jetty-maven-plugin</artifactId>
        <version>7.2.2.v20101205</version>
        <configuration>
          <scanIntervalSeconds>10</scanIntervalSeconds>
          <webAppConfig>
            <jettyEnvXml>${basedir}/src/main/resources/jetty-env.xml</jettyEnvXml>
          </webAppConfig>
        </configuration>
      </plugin>
   
    </plugins>
  </build>
</project>
```

I am using version 7.2.2 of Jetty server in the jetty-maven-plugin, also note that i configured a jetty-env.xml file which defines the JMS resources in the JNDI context. This is how my jetty-env.xml file looks like

```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" 
"http://jetty.mortbay.org/configure.dtd">
<Configure id='jms-webapp-wac' class="org.eclipse.jetty.webapp.WebAppContext">
  <New id="connectionFactory" class="org.eclipse.jetty.plus.jndi.Resource">
    <Arg>
      <Ref id='jms-webapp-wac' />
    </Arg>
    <Arg>jms/ConnectionFactory</Arg>
    <Arg>
      <New class="org.apache.activemq.ActiveMQConnectionFactory">
        <Arg>tcp://localhost:61616</Arg>
      </New>
    </Arg>
  </New>
  <New id="fooQueue" class="org.eclipse.jetty.plus.jndi.Resource">
    <Arg>jms/FooQueue</Arg>
    <Arg>
      <New class="org.apache.activemq.command.ActiveMQQueue">
        <Arg>FOO.QUEUE</Arg>
      </New>
    </Arg>
  </New>
</Configure>
```

The jetty-env.xml file defines 2 resources on is the ActiveMQConnectionFactory and second is the ActiveMQQueue. After that i did declare the messaging related resources in web.xml, so my web.xml file looks like this

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4" xmlns="http://java.sun.com/xml/ns/j2ee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee 
         http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">  
  <display-name>HelloEmbeddedServer</display-name>
  <servlet>
    <servlet-name>MessagePublishingServlet</servlet-name>
    <servlet-class>com.webspherenotes.jms.MessagePublishingServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>MessagePublishingServlet</servlet-name>
    <url-pattern>/MessagePublishingServlet/*</url-pattern>
  </servlet-mapping>
  
  
  <resource-ref>
    <description>JMS Connection</description>
    <res-ref-name>jms/ConnectionFactory</res-ref-name>
    <res-type>javax.jms.ConnectionFactory</res-type>
    <res-auth>Container</res-auth>
    <res-sharing-scope>Shareable</res-sharing-scope>
  </resource-ref>
  
  <message-destination-ref>
    <message-destination-ref-name>jms/FooQueue</message-destination-ref-name>
    <message-destination-type>javax.jms.Queue</message-destination-type>
    <message-destination-usage>Produces</message-destination-usage>
    <message-destination-link>jms/FooQueue</message-destination-link>
  </message-destination-ref>
 </web-app>
 ```
 
This is how my MessagePublishingServlet.java looks like

```java
package com.webspherenotes.jms;

import java.io.IOException;

import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.JMSException;
import javax.jms.MessageProducer;
import javax.jms.Queue;
import javax.jms.Session;
import javax.jms.TextMessage;
import javax.naming.InitialContext;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MessagePublishingServlet extends HttpServlet{
  Logger logger = LoggerFactory.getLogger(MessagePublishingServlet.class);
  Connection connection;
  Queue queue;
  

  @Override
  public void init() throws ServletException {
    logger.debug("Entering MessagePublishingServlet.init()");
    try {
      InitialContext context = new InitialContext();
      ConnectionFactory connectionFactory = (ConnectionFactory)context.lookup("java:comp/env/jms/ConnectionFactory");
      logger.debug("Connection Factory " + connectionFactory);
      connection = connectionFactory.createConnection();
      queue =(Queue) context.lookup("jms/FooQueue");
      logger.debug("After looking up the queue " + queue); 
    } catch (Exception e) {
      logger.error("Error occured in MessagePublishingServlet.init() " + e.getMessage(),e);
    }
    logger.debug("Exiting MessagePublishingServlet.init()");

  }

  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp)
      throws ServletException, IOException {
    logger.debug("Entering MessagePublishingServlet.doGet()");
    resp.setContentType("text/html");
    resp.getWriter().println("Hello from MessagePublishingServlet.doGet()");
    try {

      Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
      TextMessage textMessage = session.createTextMessage();
      textMessage.setText(req.getParameter("message"));
      MessageProducer queueSender = session.createProducer(queue);
      queueSender.send(textMessage); 
    } catch (JMSException e) {
      logger.error("Error occured in MessagePublishingServlet.doGet() " + e.getMessage(),e);

    }
    logger.debug("Exiting MessagePublishingServlet.doGet()");
  }

}
```


The init() method looks up the JMS objects from the InitialContext, in the doGet() method i am reading the value of message query parameter and using it to send a TextMessage.
