Using ATHENA-FDW with a Hadoop Sandbox VM
=========================================

## Overview ##

This HadoopFDW extension is built for the Linux distribution of [PostgreSQL by BigSQL]
(http://bigsql.org).  It allows access to Hive from PostgreSQL.

This document elucidates the steps needed to run it against *CDH 5.5
Quickstart VM* and *HDP 2.4.0 on Hortonworks Sandbox VM*.

## Pre-Requisites ##

Our latest testing has been with CDH 5.5 and HDP 2.4 from Linux using Java 8.  
For this document, we assume that you have downloaded and installed one of the following VMs:

- *CDH 5.5 Quickstart VM*
  (from [Cloudera Quickstart Downloads](http://www.cloudera.com/downloads/quickstart_vms.html))

- *HDP 2.4 on Hortonworks Sandbox VM*
  (from [Hortonworks Sandbox Downloads](http://hortonworks.com/downloads/#sandbox))

We also assume that your Hadoop VM is accessible to the machine running
PostgreSQL with the name **athena-vm** and that the host running
PostgreSQL can connect to **athena-vm** on Hive TCP port 10000 as well
as SSH TCP port 22.

Last, we assume that the host running PostgreSQL has JDK 8 installed.

### Copy the Hive Client JARs ###

First, please determine the JAR files that you will need to copy from
the Hadoop VM.

Connect to the VM using SSH from the PostgreSQL host:

```bash
ssh root@athena-vm
```
## JAR files for CDH ##

Determine the specific versions of the JAR files by running the `ls`
commands with the glob patterns shown below:

```bash
[hive@sandbox ~]$ ls /usr/lib/athena/athena-common*[0-9].jar
/usr/lib/athena/athena-common-2.6.0-cdh5.5.0.jar
[hive@sandbox ~]$ ls /usr/lib/hive/lib/hive*jdbc*[0-9]*standalone.jar
/usr/lib/hive/lib/hive-jdbc-1.1.0-cdh5.5.0-standalone.jar
```

Please note that the pattern `athena-common*[0-9].jar` precludes the
file `athena-common*-test.jar` from appearing and that the second
pattern precludes the symlink `hive-jdbc-standalone.jar` from appearing.

Once you have the paths for the two JAR files, SCP them to a directory
`hive-client-lib` on the PostgreSQL host that you are installing the
Hadoop FDW on.

## JAR files for HDP ##

Run the command `athena classpath` as the hive user to find the root
directory containing the athena JAR files (`/usr/hdp/2.4.0.0-169`):

```bash
[root@sandbox ~]# su - hive
[hive@sandbox ~]$ athena classpath
/usr/hdp/2.4.0.0-169/athena/conf:/usr/hdp/2.4.0.0-169/athena/lib/*:/usr/hdp/2.4.0.0-169/athena/.//*:
/usr/hdp/2.4.0.0-169/athena-hdfs/./:/usr/hdp/2.4.0.0-169/athena-hdfs/lib/*:/usr/hdp/2.4.0.0-169/hado
op-hdfs/.//*:/usr/hdp/2.4.0.0-169/athena-yarn/lib/*:/usr/hdp/2.4.0.0-169/athena-yarn/.//*:/usr/hdp/2
.4.0.0-169/athena-mapreduce/lib/*:/usr/hdp/2.4.0.0-169/athena-mapreduce/.//*::mysql-connector-java-5
.1.17.jar:mysql-connector-java-5.1.31-bin.jar:mysql-connector-java.jar:/usr/hdp/2.4.0.0-169/tez/*:/u
sr/hdp/2.4.0.0-169/tez/lib/*:/usr/hdp/2.4.0.0-169/tez/conf
```

The JAR files we need are:

```
/usr/hdp/2.4.0.0-169/
    |
    `--- athena/
         |
         `--- athena-common-2.7.1.2.4.0.0-169.jar
    |
    `--- hive/
         |
         `--- lib
              |
              `--- hive-jdbc-1.2.1000.2.4.0.0-169-standalone.jar
```

If you are trying this with a different version of HDP, you can
determine the specific versions of the JAR files by running the `ls`
commands with the glob patterns shown below:

```bash
[hive@sandbox ~]$ ls /usr/hdp/*/athena/athena-common*[0-9].jar
/usr/hdp/2.4.0.0-169/athena/athena-common-2.7.1.2.4.0.0-169.jar
[hive@sandbox ~]$ ls /usr/hdp/*/hive/lib/hive*jdbc*standalone.jar
/usr/hdp/2.4.0.0-169/hive/lib/hive-jdbc-1.2.1000.2.4.0.0-169-standalone.jar
```

Please note that the pattern `athena-common*[0-9].jar` precludes the
file `athena-common*-test.jar` from appearing.

Once you have the paths for the two JAR files, SCP them to a directory
`hive-client-lib` on the PostgreSQL host that you are installing the
Hadoop FDW on.

## Test that you are able to connect to Hive ##

On the PostgreSQL host, create a small JDBC program
**HiveJdbcClient.java** with the following contents:

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class HiveJdbcClient {

    private static final String url      = "jdbc:hive2://athena-vm:10000";
    private static final String user     = "";
    private static final String password = "";
    private static final String query    = "SHOW DATABASES";

    private static final String driverName = "org.apache.hive.jdbc.HiveDriver";

    public static void main(String[] args) throws SQLException {

        try {
            Class.forName(driverName);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            System.exit(1);
        }

        Connection con = DriverManager.getConnection(url, user, password);
        Statement stmt = con.createStatement();

        System.out.println("Running: " + query);
        ResultSet res = stmt.executeQuery(query);

        while (res.next()) {
            System.out.println(res.getString(1));
        }
    }
}
```

In a shell or Command Prompt, compile it with:

```sh
javac HiveJdbcClient.java
```

### Linux ###

Assuming that you copied the Hive client JAR files to the directory
`/opt/athena/hive-client-lib`, run the following command in your
shell to execute the program:

```sh
java -cp .:$(echo /opt/athena/hive-client-lib/*.jar | tr ' ' :) HiveJdbcClient
```

### Confirm Output ###

No matter the platform you ran the program on, please confirm the output
for the program as shown below:

#### CDH ####

The last two lines of the output should be:

```
Running: SHOW DATABASES
default
```

#### HDP ####

The last three lines of the output should be:

```
Running: SHOW DATABASES
default
xademo
```

## Install Hadoop-FDW ##

This FDW is included in the [PostgreSQL by BigSQL](http://bigsql.org)
distribution.  All you have to do is follow the sections below.

If you are installing from source, please follow the instructions in
[BUILD.md](BUILD.md).

## Prepare the Environment ##

The Hadoop FDW needs the following environment variable set for the PostgreSQL
server:

- **ATHENA_FDW_CLASSPATH**

    The `CLASSPATH` referencing all the Hive client JAR files we
    identified previously and the Hadoop_FDW.jar file which resides
    in the same directory as that of the PostgreSQL extension
    library files.

We provide examples for these variables below.

### Perform Setup ###

Please stop the PostgreSQL server if it is running and follow the steps
for the platform matching that of your PostgreSQL server:

We assume that `Hadoop_FDW.jar` resides under the PostgreSQL extension
library directory `/usr/local/pgsql/lib` and that the Hive client JAR
files were copied under the directory `/opt/athena/hive-client-lib`.

Assuming the JDK install location `/opt/jdk/x64/jdk1.8.0_40/`, please
run in a shell:

```sh
sudo ln -s /opt/jdk/x64/jdk1.8.0_40/jre/lib/amd64/server/libjvm.so /usr/local/pgsql/lib/libjvm.so
```

Also, in the shell, set up the requisite environment variables. In this
example, we are using bash:

```bash
export ATHENA_FDW_CLASSPATH=/usr/local/pgsql/lib/Hadoop_FDW.jar:$(echo /opt/athena/hive-client-lib/*.jar | tr ' ' :)
```

-Then start the PostgreSQL server from this shell to have the server pick
-up the variables we set up.
-
-## Use with Sample TABLE ##

The sample TABLE `sample_07` ships as part of the HDP Sandbox VM.  If
you are using CDH, we assume you have created a similar TABLE.  Below,
we show you how to access it from psql as superuser:

```sql

CREATE EXTENSION athena_fdw;

CREATE SERVER athena_server FOREIGN DATA WRAPPER athena_fdw
  OPTIONS (HOST 'athena-vm', PORT '10000');

CREATE USER MAPPING FOR PUBLIC SERVER athena_server;

CREATE FOREIGN TABLE sample_07 (
    code                   TEXT,
    description            TEXT,
    total_emp              INT,
    salary                 INT
) SERVER athena_server OPTIONS (TABLE 'sample_07');

```

With the `FOREIGN TABLE` in place, run a query against it to retrieve
data from Hive:

```sql
postgres=# SELECT code, total_emp FROM sample_07 ORDER BY code LIMIT 3;
  code   | total_emp
---------+-----------
 00-0000 | 134354250
 11-0000 |   6003930
 11-1011 |    299160
(3 rows)
```
```

Then start the PostgreSQL server from this shell to have the server pick
up the variables we set up.

