dbexample
---------

Example how to access a DataSource using OSGi services and jdbc.

Build
-----
mvn clean install

Installation
------------

feature:repo-add pax-jdbc 1.3.0
feature:install scr transaction jndi pax-jdbc-h2 pax-jdbc-pool-dbcp2 pax-jdbc-config

cat https://raw.githubusercontent.com/cschneider/Karaf-Tutorial/master/db/org.ops4j.datasource-person.cfg | tac etc/org.ops4j.datasource-person.cfg
install -s mvn:net.lr.tutorial.karaf.db/db-examplejdbc/1.0-SNAPSHOT

Test
----

The example creates and populates a table on startup, queries it and outputs the result

The output should look like this:
-------------------------------------------
Using datasource H2, URL jdbc:h2:~/test
Christian Schneider, @schneider_chris,
-------------------------------------------
