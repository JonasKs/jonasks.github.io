---
title:  "Wrapping my head around OpenShift User Magic"
date:   2019-03-17 15:04:23
categories: [tech]
tags: [tech, openshift, mssql, drivers]
---
Docker tries to solve the shipping problem by ensuring that no matter where a container is run, the environment will always be the same. 
In other words, if you build your Docker image on a Windows machine, it will run equally on a Linux machine and on AWS. OpenShift breaks this concept by adding an extra security layer on top of the container, 
making things often not work in OpenShift even if it works locally. This short article will describe how I wrapped my head around this when dealing with a MSsql driver. 

OpenShift adds a random generated pool of userIDs to your project. If you want to check out your assigned user IDs, write `oc describe project <my_project>`. 
```python
➜  my_project git:(develop) ✗ oc describe project my_project
Name:                   my_project
Created:                5 months ago
Labels:                 <none>
Annotations:            openshift.io/requester=hotfix
                        openshift.io/sa.scc.supplemental-groups=1001010000/10000
                        openshift.io/sa.scc.uid-range=1001010000/10000
```

If you need more permissions on your folders or files, you need to specifically do this through `chmod` and `chown` and assign it to the user 1001. This user is somehow "magic", meaning what ever permissions you give it, will work for any user your container gets assigned. 
```python
RUN     chown -R 1001:0 /app/src  && chmod -R ug+rwx /app/src
```
Remember to set the user as `1001` too. 
```python
USER    1001
```

This works fine in most cases, but when trying to connect to a Microsoft SQL server, the ODBC driver (`ODBC Driver 17 for SQL Server`) crashes.

```python
>>> import pyodbc
>>> db = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};SERVER=SERVER_ADDRESS,1433;DATABASE=database;UID=USERNAME;PWD=PASSWORD;")
('IM004', "[IM004] [unixODBC][Driver Manager]Driver's SQLAllocHandle on SQL_HANDLE_HENV failed (0) (SQLDriverConnect)")
```

Since the error don't give us much (neither does google), recording the stack trace and doing a deeper dive was the solution for me. 

By issuing the command `strace -t -f -o /home/test.txt isql -v <IP> <username> <password>` and reading through all the thousands of lines I could see every bit of the operating system the driver tried to access. 
At the very end, just before it crashes you can see it opens the `/etc/passwd` file: `open("/etc/passwd", O_RDONLY|O_CLOEXEC) = 3`. 
If you look in this file, you can see that there is no user called `1001`, or `1001010000`. Typing `whoami` confirms this:

```python
I have no name!@c32ea0cc530e:/app/src$ whoami
whoami: cannot find name for user ID 1001010000
```

This means we will have to add this ourselves. On container startup, check userID, add it to the `/etc/passwd` file and then run the service (in my case a Django webserver) through a 
simple bash script: 
```python
#!/usr/bin/env bash
# Ensure that assigned uid has entry in /etc/passwd.

if [ `id -u` -ge 1000 ]; then
    cat /etc/passwd | sed -e "s/^$NB_USER:/builder:/" > /tmp/passwd
    echo "$NB_USER:x:`id -u`:`id -g`:,,,:/home/$NB_USER:/bin/bash" >> /tmp/passwd
    cat /tmp/passwd > /etc/passwd
    rm /tmp/passwd
fi

# Run Django gunicorn
gunicorn my_project.wsgi:application --bind 0.0.0.0:8080
```

We are now able to connect to the MSsql database, as proven with the following Python code:

```python
>>> import pyodbc
>>> db = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};SERVER=SERVER_ADDRESS,1433;DATABASE=database;UID=USERNAME;PWD=PASSWORD;")
>>> cursor = db.cursor()
>>> cursor.execute("SELECT @@version;")
<pyodbc.Cursor object at 0x7fdb73c35a80>
>>> row = cursor.fetchone()
>>> while row:
...     print(row[0])
...     row = cursor.fetchone()
...
Microsoft SQL Server 2017 (RTM-CU11) (KB4462262) - 14.0.3038.14 (X64)
        Sep 14 2018 13:53:44
        Copyright (C) 2017 Microsoft Corporation
        Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)
```