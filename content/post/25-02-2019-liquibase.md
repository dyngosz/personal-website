---
title:  "Leveraging Liquibase to keep your database neat"
date:   2019-02-25 15:00:00
tags: ["java", "tech", "tools"]
draft: false
---

What would come to your mind if I'd asked you to imagine **utter chaos** in any IT project taking place nowadays? I think it's safe to assume that the majority of people might think about something like *awfully bad codebase* or *lack of any tool to track tasks* and those would be very accurate thoughts indeed.

But another way to properly describe chaos within a project, especially one dealing with storing and handling any kind of data, might be a vast and constantly expanding database with no possibility to track its transformations whatsoever.
That is why you should consider making friends with any **database refactoring tools** long before your *db* gets big and a developer wanting to modify its schema would have to pray not to break everything apart. 💥

In this post I want to briefly describe how Liquibase works and propose a setup in which database changelogs will be automatically generated based on JPA annotated entities within a Java project built with Maven.

### Keeping track of your database
There are two very popular tools that allow you to effectively track how your database changes over time: [Flyway](https://flywaydb.org/) and [Liquibase](https://www.liquibase.org/). They are both efficient and get their task done but for the purpose of this post I am going to stick to the latter. Liquibase was introduced in 2006 as an open-source library by a software developer [Nathan Voxland](https://twitter.com/nvoxland). The main idea behind this tool is quite simple. Whenever we change some parts of our source code, we often need to modify the database schema as well. To implement it an SQL script might get introduced to make things easier but as time goes by, the number of such scripts is likely to increase and **let's not forget** that any developer performing a release of the application has to explicitly remember to run the scripts. This workflow is very prone to error and may result in a abnormally configured database schema.
That is where Liquibase comes to the rescue. All the changes to the database are put in a text file and applied automatically by the tool upon running it. The advantage of such a solution is that all the scripts are executed at once and Liquibase knows which scripts have already been executed on the target database and it only runs those not yet complete. The metadata for all already applied scripts is then stored in a special database table. This allows us to effectively track all changes and apply any refactor with an ease.

### One changelog to rule them all
Liquibase supports migration scripts in such formats as XML, YAML and JSON. In these scripts you can define all of the changes you performed on your database like creating tables, adding columns or primary keys and many more. For the whole list of available modifications please visit tool's official [documentation](https://www.liquibase.org/documentation/changes/index.html).
When it comes to the format of your files with schema changes - *changelogs*, it is entirely up to you but I tend to stick to XML which, as oddly as it may seem, I find the most readable.
For me the solution that worked most efficiently was to **have a master changelog file** in which I would include all of specific smaller changelogs in which the changes to my database are applied. The contents of my *changelog-master.xml* could then look like this:

``` XML
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">

    <include file="db/v0001.xml"/>
    <include file="db/v0002.xml"/>
    <include file="db/v0003.xml"/>

</databaseChangeLog>
```

### Automate the hell out of it
Since we all love automating stuff (*don't we?*) I find it cumbersome to have to manually add entries in my changelog every time I change my source code and thus my database schema. When it comes to Java projects based on classic Spring Boot and Hibernate stack there is a great solution to this issue. When we are dealing with relational databases we can use *Java Persistence API* which greatly simplifies *db* management. This is in fact a standard solution if we are using Hibernate as our object-relational mapping implementation.

The most important part of the setup is configuration of Liquibase itself. To correctly leverage the tool first you need to add it to your project's build file. This can be done by adding two dependencies in *pom.xml*:

``` XML
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-maven-plugin</artifactId>
    <version>3.4.1</version>
</dependency>
```

Thanks to the *liquibase-maven-plugin* we will be able to use database refactoring tool along with our build automation to automatically generate *db* changelogs. We also need to perform additional configuration of the plugin by adding these lines to the same file:
``` XML
<build>
        <plugins>
            <plugin>
                <groupId>org.liquibase</groupId>
                <artifactId>liquibase-maven-plugin</artifactId>
                <version>3.5.3</version>
                <dependencies>
                    <dependency>
                        <groupId>org.liquibase.ext</groupId>
                        <artifactId>liquibase-hibernate4</artifactId>
                        <version>3.6</version>
                    </dependency>
                    <dependency>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring-beans</artifactId>
                        <version>4.1.7.RELEASE</version>
                    </dependency>
                    <dependency>
                        <groupId>org.springframework.data</groupId>
                        <artifactId>spring-data-jpa</artifactId>
                        <version>1.7.3.RELEASE</version>
                    </dependency>
                </dependencies>
                <configuration>
                    <propertyFile>src/main/resources/liquibase.properties</propertyFile>
                    <changeLogFile>src/main/resources/db/changelog/changelog-master.xml</changeLogFile>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

Apart from adding dependencies we also specify *propertyFile* which will serve as our Liquibase configuration entrypoint and *changeLogFile* to which the actual changes based on performed diffs will be appended. We are already finished with our *pom.xml* and can move to the second step of configuration which is the mentioned property file. Contents of this config will heavily depend on your own database solution since Liquibase abstracts away from SQL scripts completely and decouples database refactoring from the underlying database technology. In my case I use [PostgreSQL](https://www.postgresql.org/) and my *liquibase.properties* looks like this:

```pkgconfig
referenceUrl=hibernate:spring:com.dyngosz.application.entity?dialect=org.hibernate.dialect.PostgreSQLDialect
referenceDriver=liquibase.ext.hibernate.database.connection.HibernateDriver

driver=org.postgresql.Driver
url=jdbc:postgresql://localhost:5432/application
username=yourDatabaseUsername
password=yourDatabasePassword

diffChangeLogFile=src/main/resources/liquibase-diff.xml
```

What is important is that you need to properly set up driver for your database instance. In other case the solution won't work at all. To correctly perform a diff between your entities you need to configure *url*, *username* and *password* properties which refer to your database instance and are vital to the process. The *referenceUrl* points to the package within your appliction containing all defined entites which will be analyzed to create a changelog compared with your schema, in my case it is *com.dyngosz.application.entity*.

Once you have all that the configuration is complete and the application is ready to begin analyzing your new entities and creating changelogs based on them. As an example I came up with a basic implementation of users and their roles in my demo application. This setup is made out of two entities

*User.java*:
```Java
@Entity
@Table(name = "user")
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @NotNull
    private String username;

    @NotNull
    private String password;

    @Transient
    private String passwordConfirm;

    @ManyToMany
    @JoinTable(name = "user_role", joinColumns = @JoinColumn(name = "user_id"), inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles;
}
```
and *Role.java*:
```Java
@Entity
@Table(name = "role")
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @NotNull
    private String name;

    @ManyToMany(mappedBy = "roles")
    private Set<User> users;
}
```

This is a very simple configuration for a new application which will later incorporate any kind of authentication solutions like Spring Security. Every entity is properly annotated using JPA syntax. There are also some additional [Lombok](https://projectlombok.org/) annotations just to add some flavour to it. Assuming that our database is empty and we just created those two entities, next thing to do would be to add those modifications to our *changelog* file to make them visible and to create the related tables upon startup of the project.
To achieve that we just have to invoke **mvn liquibase:diff** and the *liquibase-diff.xml* is generated:
```XML
<changeSet author="wiktordyngosz" id="1550740413906-3">
        <createTable tableName="role">
            <column autoIncrement="true" name="id" type="BIGINT">
                <constraints primaryKey="true" primaryKeyName="rolePK"/>
            </column>
            <column name="name" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>

    <changeSet author="wiktordyngosz" id="1550740413906-4">
        <createTable tableName="user">
            <column autoIncrement="true" name="id" type="BIGINT">
                <constraints primaryKey="true" primaryKeyName="userPK"/>
            </column>
            <column name="password" type="VARCHAR(255)"/>
            <column name="username" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>

    <changeSet author="wiktordyngosz" id="1550740413906-5">
        <createTable tableName="user_role">
            <column name="user_id" type="BIGINT">
                <constraints nullable="false"/>
            </column>
            <column name="role_id" type="BIGINT">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <changeSet author="wiktordyngosz" id="1550740413906-6">
        <addPrimaryKey columnNames="user_id, role_id" tableName="user_role"/>
    </changeSet>
    <changeSet author="wiktordyngosz" id="1550740413906-8">
        <addForeignKeyConstraint baseColumnNames="user_id" baseTableName="user_role"
                                 constraintName="FK_apcc8lxk2xnug8377fatvbn04" deferrable="false"
                                 initiallyDeferred="false" referencedColumnNames="id" referencedTableName="user"/>
    </changeSet>
    <changeSet author="wiktordyngosz" id="1550740413906-9">
        <addForeignKeyConstraint baseColumnNames="role_id" baseTableName="user_role"
                                 constraintName="FK_it77eq964jhfqtu54081ebtio" deferrable="false"
                                 initiallyDeferred="false" referencedColumnNames="id" referencedTableName="role"/>
    </changeSet>
</databaseChangeLog>
```

Only thing left would be to move those change sets to a specific file under a name following convention like *db/v0001.xml* and include them in our master changelog as mentioned before. **Good luck with all the database refactors**
