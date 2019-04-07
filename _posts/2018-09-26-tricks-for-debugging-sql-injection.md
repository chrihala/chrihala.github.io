---
layout: post
title: Tricks for debugging SQL Injection Exploitation
date: '2018-09-26T09:23:00.002-07:00'
author: Hland
tags: 
modified_time: '2018-09-26T09:23:50.354-07:00'
---


Here are a few tricks that are useful for debugging SQL injection bugs on OS X when you have the codebase, can run the application locally and wanna see the actually queries being run in the database. My use-case for this is automating exploitation of relatively complex SQL injections with SQLMap to prove data exfiltration capabilities through a vulnerability. 

This is targeted at MySQL running on OS X and installed via HomeBrew. I like to use [Sequel Pro](https://www.sequelpro.com/) as it's nice to have some GUI support and don't have to lookup how to setup permissions and such every time you wanna setup a database. 

First, to install MySQL via HomeBrew:
{% highlight ruby %}
brew install mysql
brew services start mysql
brew services stop mysql
{% endhighlight %}



Enable query logging. Run this from a MySQL prompt:
```
set GLOBAL general_log = 'ON';
SHOW VARIABLES LIKE '%general_log%'`
```
Watch the SQL queries hit your database while you run SQLMap or some other gatling-gun style tool against your local application. 

```
tail -f /usr/local/var/mysql/<your-hostname>.log
```

