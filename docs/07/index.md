---
title: Apache Karaf Tutorial Part 7 - Camel JPA and JTA transactions
---

Practical Camel example that polls from a database table and sends the contents as XML to a jms queue. The route uses a JTA transaction to synchronize the DB and JMS transactions. An error case shows how you can handle problems.

## Route and Overview

![Image](jpa2jms.png)

```
from("jpa://net.lr.tutorial.karaf.camel.jpa2jms.model.Person").id("jpa2jms")
.onException(Exception.class).maximumRedeliveries(3).backOffMultiplier(2).handled(true).to("file:error")
.transacted()
.marshal(df)
.bean(new ExceptionDecider())
.to("jms:person");
```

The route starts with a jpa endpoint. It is configured with the fully qualified name of a JPA @Entity. From this entity camel knows the table to poll and how to read and remove the row. The jpa endpoint polls the table and creates a Person object for each row it finds. Then it calls the next step in the route with the Person object as body. The jpa component also needs to be set up separately as it needs an EntityManagerFactory.

The onException clause makes the route do up to 3 retries with backoff time increasing by factor 2 each time. If it still fails the message is passed to a file in the error directory.

The next step transacted() marks the route as transactional it requires that a TransactedPolicy is setup in the camel context. It then makes sure all steps in the route have the chance to participate in a transaction. So if an error occurs all actions can be rolled back. In case of success all can be committed together.

The marshal(df) step converts the Person object to xml using JAXB. It references a dataformat df that sets up the JAXBContext. For brevity this setup is not shown here.

The ExceptionDecidet bean allows to trigger an exception if the name of the person is error. So this allows us to test the error handling later.

The last step to("jms:person") sends the xml representation of person to a jms queue. It requires that a JmsComponent named jms is setup in the camel context.

```
from("jms:person").id("jms2log")
.transacted()
.convertBodyTo(String.class)
.to("log:personreceived");
```

This second route simply listens on the person queue, reads and displays the content. In a production system this part would tpyically be in another module.

## Person as JPA Entity JAXB class

The Person class acts as a JPA Entity and as a JAXB annotated class. This allows us to use it in the camel-jpa component as well as during the marshalling. Keep in mind though that this
would rather be a bad practice in production as it would tie the DB model and the format of the JMS message together. So for real integrations it would be better to have separate beans for JPA and JAXB and do a manual
conversion between them.

```
@Entity
@XmlType
@XmlRootElement
public class Person {
    private String name;
    private String twitterName;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getTwitterName() { return twitterName; }
    public void setTwitterName(String twitterName) { this.twitterName = twitterName; }
}
```

## DataSource and ConnectionFactory setup

We use an XADataSource for Derby (See https://github.com/cschneider/Karaf-Tutorial/blob/master/db/datasource/datasource-derby.xml). As the default ConnectionDactory provided by ActiveMQ in Karaf is not XA ready we define the broker and ConnectionFactory definition by hand (See https://github.com/cschneider/Karaf-Tutorial/blob/master/cameljpa/jpa2jms/localhost-broker.xml). Together with the Karaf transaction feature these provide the basis to have JTA transactions.

JPAComponent, JMSComponent and transaction setup
An important part of this example is to use the jpa and jms components in a JTA transaction. This allows to roll back both in case of an error.
Below is the blueprint context we use. We setup the JMS component with a ConnectionFactory referenced as an OSGi service.
The JPAComponent is setup with an EntityManagerFactory using the jpa:unit config from Aries JPA. See Apache Karaf Tutorial Part 6 - Database Access for how this works.
The TransactionManager proviced by Aries transaction is referenced as an OSGi service, wrapped as a spring PlattformTransactionManager and injected into the JmsComponent and JPAComponent.

```
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cm="http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0"
    xmlns:jpa="http://aries.apache.org/xmlns/jpa/v1.1.0"
	xsi:schemaLocation="
  http://www.osgi.org/xmlns/blueprint/v1.0.0 http://www.osgi.org/xmlns/blueprint/v1.0.0/blueprint.xsd
  http://www.osgi.org/xmlns/blueprint-ext/v1.1.0 https://svn.apache.org/repos/asf/aries/tags/blueprint-0.3.1/blueprint-core/src/main/resources/org/apache/aries/blueprint/ext/blueprint-ext.xsd
  http://camel.apache.org/schema/blueprint http://camel.apache.org/schema/blueprint/camel-blueprint.xsd
  http://aries.apache.org/xmlns/jpa/v1.1.0 http://aries.apache.org/schemas/jpa/jpa_110.xsd
  ">

    <reference id="connectionFactory" interface="javax.jms.ConnectionFactory" />

    <bean id="jmsConfig" class="org.apache.camel.component.jms.JmsConfiguration">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="transactionManager" ref="transactionManager"/>
    </bean>

    <bean id="jms" class="org.apache.camel.component.jms.JmsComponent">
        <argument ref="jmsConfig"/>
    </bean>

    <reference id="jtaTransactionManager" interface="javax.transaction.TransactionManager"/>

    <bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
        <argument ref="jtaTransactionManager"/>
    </bean>

    <bean id="jpa" class="org.apache.camel.component.jpa.JpaComponent">
        <jpa:unit unitname="person2" property="entityManagerFactory"/>
        <property name="transactionManager" ref="transactionManager"/>
    </bean>

    <bean id="jpa2jmsRoute" class="net.lr.tutorial.karaf.camel.jpa2jms.Jpa2JmsRoute"/>

    <bean id="PROPAGATION_REQUIRED" class="org.apache.camel.spring.spi.SpringTransactionPolicy">
        <property name="transactionManager" ref="transactionManager"/>
     </bean>

    <camelContext id="jpa2jms" xmlns="http://camel.apache.org/schema/blueprint">
        <routeBuilder ref="jpa2jmsRoute" />
    </camelContext>

</blueprint>
```

## Running the Example

You can find the full example on github : JPA2JMS Example
Follow the Readme.txt to install the necessary Karaf features, bundles and configs.

Apart from this example we also install the dbexamplejpa. This allows us to use the person:add command defined there to populate the database table.
Open the Karaf console and type:

```
person:add "Christian Schneider" @schneider_chris
log:display
```

You should then see the following line in the log:

```
2012-07-19 10:27:31,133 | INFO  | Consumer[person] | personreceived ...
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person>
    <name>Christian Schneider</name>
    <twitterName>@schneider_chris2</twitterName>
</person>
```

## So what happened

We used the person:add command to add a row to the person table. Our route picks up this record, reads and converts it to a Person object. Then it marshals it into xml and sends to the jms queue person.
Our second route then picks up the jms message and shows the xml in the log.

## Error handling

The route in the example contains a small bean that reacts on the name of the person object and throws an exception if the name is "error".
It also contains some error handling so in case of an exception the xml is forwarded to an error directory.

So you can type the following in the Karaf:

```
person:add error error
log:display
```

This time the log should not show the xml. Instead it should appear as a file in the error directory below your karaf installation.

## Summary

In this tutorial the main things we learned are how to use the camel-jpa component to write as well as to poll from a database and how to setup and use jta transactions to achieve solid error handling.

[Back to Karaf Tutorials](..)
