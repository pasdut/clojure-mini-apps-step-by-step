+++
title = "SQL Inspector : Clojure + JDBC + JavaFX Desktop application - Part 1"
author = ["Pascal Dutilleul"]
draft = false
+++

## What? {#what}

"SQL Inspector" is a desktop application written in Clojure.  It shows all tables of a database. There is a textbox that you can use to filter the tables. Selecting a table will show all its columns and types.
![](/si-sketch-v2.png)


## Setup and configure Database Container {#setup-and-configure-database-container}

This is a database application, thus we need a database.  I'll go for MS SQL Server because that is the database I use on my current project.

Fortunately there is a Docker container [here](https://hub.docker.com/_/microsoft-mssql-server) from Docker Hub so it will run on linux too. Check the install info at: [Docker: Install containers for SQL Server on Linux - SQL Server | Microsoft Docs](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-ver15&pivots=cs1-bash).

The following shell script will launch the docker container.

```bash
sudo docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=<YourStrong@Passw0rd>" \
   -p 1433:1433 --name sqlserver2019 -h sqlserver2019 \
   -v sql2019data:/var/opt/mssql \
   -d mcr.microsoft.com/mssql/server:2019-latest
```

Have a look at the parameter _-v_.  This says to create a new volume called _sql2019data_ (if it doesn't already exists). This is actually a container that persists the data and is mounted in the _sqlserver2019_ on the folder _var/opt/mssql_.  If you kill the _sqlserver2019_ container you won't lose the actual data of the database.  This is because the data is not stored in the regular container _sqlserver2019_ but in the container _sql2019data_. See [Use volumes | Docker Documentation](https://docs.docker.com/storage/volumes/) and also [Configure and customize SQL Server Docker containers - SQL Server | Microsoft...](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-docker-container-configure?view=sql-server-ver15&pivots=cs1-bash#persist)

Next steps in the database initialization is:

-   Create a new database _test_db_.
-   Create a new user _test_user_.
-   Grant the needed permissions to the _test_user_. To be honest, I only found out what permissions were needed once I started to create the tables in Step 02 here below.
-   Change the password for user _sa_.  The original password is forever saved in the environment variable SA_PASSWORD in the containier. Which means it is visible for anyone with access to the container. So it is very important to change the password for _sa_.

For this we will launch the _sqlcmd_ application from within the container :

```bash
sudo docker exec -it -w /opt/mssql-tools/bin sqlserver2019 ./sqlcmd -S localhost -U SA
```

And enter the following commands in the sqlcmd (each command will be executed when you typed 'GO' on a new line):

```sql
1> create database test_db
2> go
1> create login test_user with password = 'test123!'
2> go
1> use test_db
2> go
Changed database context to 'test_db'.
1> create user test_user for login test_user
2> go
1> exec sp_addrolemember 'db_datareader', 'test_user'
2> go
1> grant alter on schema::dbo to test_user
2> go
1> grant create table to test_user
2> go
1> grant references to test_user
2> go
1> alter login sa with password = 'NewPassw0rd!'
2> go
1> quit
```

And finally, when we've finished playing with the container we must stop it. And equally important - because we gave the container a name -  we must also remove the container. If we don't delete,  we won't be able to restart the container the next time.

```bash
    sudo docker stop sqlserver2019
    sudo docker rm sqlserver2019
```


## Practical stuff {#practical-stuff}

You can find the code at <https://github.com/pasdut/sqlinspector/>.
Every step has its own branch.
If you want to follow along there are a couple of possibilities, depending on your own personal preferences:


### Clone everything and switch branches on the fly {#clone-everything-and-switch-branches-on-the-fly}

```bash
# clone everything
~$ git clone https://github.com/pasdut/sqlinspector.git
Cloning into 'sqlinspector'...
~$ cd sqlinspector/

# look at the core.clj from step02
~/sqlinspector$ git switch step02
Branch 'step02' set up to track remote branch 'step02' from 'origin'.
Switched to a new branch 'step02'
~/sqlinspector$ cat src/sqlinspector/core.clj
# or you could run step02
~/sqlinspector$ lein run

#switch back to main to see the latest version of core.clj
~/sqlinspector$ git switch main
Switched to branch 'main'
Your branch is up to date with 'origin/main'.
~/sqlinspector$ cat src/sqlinspector/core.clj
```


### Clone each step in its own folder {#clone-each-step-in-its-own-folder}

Useful if you want to get all steps in its own folder. You can then open for instance all _core.clj_ files of all different steps at once in your favorite editor.

```bash
# get step01 in its own folder
~$ mkdir step01
~$ cd step01
~/step01$ git clone -b step01 https://github.com/pasdut/sqlinspector.git
Cloning into 'sqlinspector'...
~/step01$ cd ..

# get step02 in its own folder
~$ mkdir step02
~$ cd step02
~/step02 git clone -b step02 https://github.com/pasdut/sqlinspector.git
Cloning into 'sqlinspector'...
~/step02$ cd ..

# now the source of both versions are available at the same time
~$ cat ~/step01/sqlinspector/src/sqlinspector/core.clj
~$ cat ~/step02/sqlinspector/src/sqlinspector/core.clj
```


### Use the github website {#use-the-github-website}

Click on the _branches_ icon to see all branches.
[![](/ox-hugo/si-gh-01-s.png)](/ox-hugo/si-gh-01.png)

You can then make a branch active by clicking on it, in the example below click on the _step02_ branch.
[![](/ox-hugo/si-gh-02-s.png)](/ox-hugo/si-gh-02.png)

Now you'll see the _step02_ branch is selected.  Click on the commits to see the commits.
[![](/ox-hugo/si-gh-03-s.png)](/ox-hugo/si-gh-03.png)

The latest commit is the one that is actually added in this step.  If you click on the commit hash you'll see the diff between previous commits.
[![](/ox-hugo/si-gh-04-s.png)](/ox-hugo/si-gh-04.png)

Green is what has been added, red is what has been removed. In the example below you'll notice the single line _:dependencies_ is replaced by a _:dependencies_ that now spans multiple lines.
[![](/ox-hugo/si-gh-05-s.png)](/ox-hugo/si-gh-05.png)


## Step 01 - Create application {#step-01-create-application}

Let's create a new application via [Leiningen](https://leiningen.org/).  And while we're here, let's also immediately test if the application runs. Just to be sure everything is correctly installed.

```nil
~$ lein new app sqlinspector
~$ cd sqlinspector
~/sqlinspector$ lein run
Hello, World!
```


## Step 02 - JDBC Database connection experimentation in the REPL {#step-02-jdbc-database-connection-experimentation-in-the-repl}

Data access is via [Java JDBC API](https://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/). And on top of JDBC we have the clojure library [next-jdbc](https://github.com/seancorfield/next-jdbc).
First of all we must include the dependencies for [next.jdbc](https://github.com/seancorfield/next-jdbc) and the [JDBC driver for MS SQL](https://docs.microsoft.com/en-us/sql/connect/jdbc/download-microsoft-jdbc-driver-for-sql-server?view=sql-server-ver15).  There is no need to manually download the JDBC driver, _lein_ will take care of that for us.

```clojure
  :dependencies [[org.clojure/clojure "1.10.1"]
                 [com.github.seancorfield/next.jdbc "1.2.772"]
                 [com.microsoft.sqlserver/mssql-jdbc "10.2.0.jre11"]]
```

When we start a REPL these new dependencies will now be retrieved:

```nil
~/sqlinspector$ lein repl
nREPL server started on port 40705 on host 127.0.0.1 - nrepl://127.0.0.1:40705
REPL-y 0.4.4, nREPL 0.7.0
Clojure 1.10.1
OpenJDK 64-Bit Server VM 11.0.11+9-Ubuntu-0ubuntu2.20.10
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

sqlinspector.core=> (quit)
Bye for now!
```

From now on, I will no longer use the command line to launch a REPL.

Instead I will fully embrace [REPL driven development](https://practical.li/clojure-staging/repl-driven-devlopment.html) and enter the expressions directly in my editor (I use emacs + Cider, but you are of course free to use IntelliJ + Cursive or VSCode + Calva)

Let's prepare the _core.clj_ by requiring _next.jdbc_ and adding an empty _comment_ block where we can enter code to be executed immediately.

```clojure
(ns sqlinspector.core
  (:gen-class)
  (:require [next.jdbc :as jdbc])
  )

(defn -main
  "I don't do a whole lot ... yet."
  [& args]
  (println "Hello, World!"))

;;----------------------------------------------------------------------------------------
;; Below is a big chunk of . This is used to enter expressions in the REPL directly
;; from within the Editor.
;;----------------------------------------------------------------------------------------
(comment

  ;;type here below all the expressions you want to evaluate in the repl

  )
```

If we now send the complete file to the repl, the jdbc library will be loaded and we can start using it.

Let's see if we can execute a _select_. Remember, just type the code in the comment section and instruct your editor to send the expression to the REPL.

```clojure
;;----------------------------------------------------------------------------------------
;; Below is a big chunk of comment. This is used to enter expressions in the REPL directly
;; from within the Editor.
;;----------------------------------------------------------------------------------------
(comment

  ;; the connection parameters
  (def db {:dbtype "sqlserver"
           :user "test_user"
           :password "test123!"
           :host "127.0.0.1"
           :encrypt false
           :dbname "test_db"})
  (def ds (jdbc/get-datasource db))

  ;; test with the most simple select
  (jdbc/execute! ds ["select 123 as just_a_number"])

  )
```

Hooray, it works! We just give _jdbc/execute!_ a query and it will be executed.

Let's beef up our database with some tables :

```clojure
  (jdbc/execute! ds [(str "create table t_customer ( \n"
                          "  id_customer int not null identity(1,1) \n"
                          "    constraint pk_t_customer primary key, \n"
                          "  first_name varchar(250), \n"
                          "  last_name varchar(250), \n"
                          "  last_modified datetime not null \n"
                          "    constraint df_t_customer default (getdate()))")])

  (jdbc/execute! ds [(str "create table t_address_type ( \n"
                          "  address_type varchar(50) not null \n"
                          "    constraint pk_t_address_type primary key, \n"
                          "  info varchar(250))")])

  (jdbc/execute! ds [(str "create table t_customer_address ( \n"
                          "  id_customer_address int not null identity(1,1) \n"
                          "    constraint pk_t_customer_address primary key, \n"
                          "  id_customer int not null \n"
                          "    constraint fk_t_customer_address__customer \n"
                          "    foreign key references t_customer(id_customer), \n"
                          "  address_type varchar(50) not null \n"
                          "    constraint fk_t_customer_address__addres_type \n"
                          "    foreign key references t_address_type(address_type), \n"
                          "  is_default bit not null \n "
                          "    constraint df_t_customer_address default (0), \n"
                          "  info varchar(250))")])
```

And now the real work : extract the table names from the database. In Sql Server we find this info in _sys.tables_.

```clojure
  (jdbc/execute! ds [(str "select name as table_name, create_date, modify_date \n"
                          "from sys.tables order by name") ])
```

We can find the columns of a given table in _sys.columns_:

```clojure
(let [table-name "t_customer"]
    (jdbc/execute! ds
                   [(str "select \n"
                         "  c.column_id, \n"
                         "  c.name as column_name, \n"
                         "  t.[name] as type_name, \n"
                         "  c.max_length, \n"
                         "  c.is_nullable, \n"
                         "  c.is_identity \n"
                         "from sys.columns c \n"
                         "join sys.types t on t.system_type_id = c.system_type_id \n"
                         "where c.[object_id] = object_id(?) \n"
                         "order by c.column_id")
                    table-name]))
```


## Step 03 - Create the database related functions {#step-03-create-the-database-related-functions}

Let's remove the code from the REPL experiment from _Step 02_ and carve it in stone instead:

-   The _(def db ...)_ and _(def ds ...)_ are moved out of the _(comment ...)_ part into the actual source of _core.clj_.
-   And two new functions _retrieve-all-tables_ and _retrieve-table-columns_ are created.

Step 03 is just a formality, there is no new code.  This step is just created so you can see on the branch how the code is moved out of the _(comment ...)_ block.

Test in the REPL if the new functions work as expected.

```clojure
  ;; check if the new functions work as expected
  (retrieve-all-tables)
  (retrieve-table-columns "t_customer")
```


## End of Part 1 {#end-of-part-1}

This finishes the end of the first Part.  Now we are able to get the needed data from the database. In Part 2 we will start with the UI
