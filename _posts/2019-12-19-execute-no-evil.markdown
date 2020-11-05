---
layout: post
title: CTF Writeup - Execute No Evil - X-mas CTF 2019
date: 2019-12-19 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: xmas-ctf.png # Add image post (optional)
tags: [ctf, web security] # add tag
---

An SQL Injection web-challenge in X-mas CTF 2019 solved with the xSTF CTF Team.

From the ground up we know that it's an SQL Injection challenge due to the intro:

![Intro](https://i.imgur.com/vl42YbS.png)

When we open the page we have a web application that queries a database and prints an user from the said database.

![Page](https://i.imgur.com/GT9oZJb.png)

When we inspect the source code of the page, we can see this:

![Source-code](https://i.imgur.com/VtDK0S8.png)

When we reach the endpoint identified, we can see the PHP source code used to query the database.

As the introduction told us, the injection point is inside a multi line comment - see below:

![Php-db-query](https://i.imgur.com/dr70gD2.png)

My first thought would obviously be to feed a payload that would close the multi-line comment ( \*/ ) and proceed with the injection from there.

If we see the source-code again, we can see that this is impossible, since the input variable is stripped from asterisks (\*) with str_replace("*","",$name)

After banging my head against the wall for about 30 minutes, I decided to check out MySQL's documentation regarding multi-line comments and found the key to this challenge:

![Mysql documentation](https://i.imgur.com/T8px4tD.png)

As we can see, MySQL has a feature advertised as a "portability" feature that allows SQL Code to be executed inside comments.

The point of this feature is to use it to write specific MySQL code and it would only be parsed if the RDBMS engine is MySQL while other engines (like oracle, postgres, etc.) would ignore it because it's commented out.

This means we can inject SQL code inside the comment and make it run by starting our input with a "!"


To test this, I injected the following payload:
```
!'Geronimo' UNION ALL SELECT 1,2 FROM users WHERE name=
```

![SQL_Payload1](https://i.imgur.com/ELmTi5m.png)

This gave an error that meant the SQL Query was not working properly.


I debugged it in a local MySQL instance on my machine and realised that this was probably happening because a mismatch of column number (which is required for an SQL UNION)

If I could have injected an  * in there, I could've tested it much quickly because I would for sure match the number of columns 

if I used the same table, but again str_replace() prevented this.


I tried my payload manually with 3 columns instead and boom! We got a hit

```
!'Geronimo' UNION ALL SELECT 1,2,3 FROM users WHERE name=
```

![SQL_Payload2](https://i.imgur.com/TUfiHlL.png)

Ok, now we know that the SQL code we inject here works. It's time to enumerate the DB and find the flag.

I used MySQL Information_Schema table to enumerate which tables there were in the database with the following payload:

``` 
!'Geronimo' UNION ALL SELECT 1,table_name,3 from Information_Schema.tables UNION ALL SELECT 1,2,3 from users where name=
```
![SQL_Payload3](https://i.imgur.com/KDselPm.png)

Ok there's our flag table. As we can not use * to print all the columns in that table, we still have to enumerate the column names of it in order to build a functioning query.

For that, once again we use MySQL's Information Schema, but this time reaching out for the column's name:

```
!'Geronimo' UNION ALL SELECT 1,column_name,3 FROM Information_Schema.columns WHERE table_name='flag' UNION ALL SELECT 1,2,3 from users where name=
```

![SQL_Payload4](https://i.imgur.com/dj0cOBS.png)

Now we have our flag table and our column, it's time for our final payload

```
!'Geronimo' UNION ALL SELECT 1,whatsthis,3 FROM flag UNION ALL SELECT 1,2,3 from users where name=
```

![SQL_Payload5](https://i.imgur.com/3GyqOGC.png)

### Flag:

```
X-MAS{What?__But_1_Th0ught_Comments_dont_3x3cvt3:(}
```
