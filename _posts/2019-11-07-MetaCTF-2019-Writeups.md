---
title: MetaCTF 2019 Writeups
author: Andy Smith
---

# Web Exploitation
## C3 Internal Knowledge Base
![c3_employee_knowledgebase.png](/assets/images/metactf/c3_employee_knowledgebase.png)
In this problem we're given a login page and need to access the internal wiki. Trying to browse to the wiki gives a 302 Found redirect with a header `Location: ./?error=unauthorized&source=./knowledgebase.php`. Chrome happily follows this redirect and does not display any information in the response body, however if we use a command line tool like `curl` we can just ignore this redirect. Running `curl https://problems.metactf.com/content/logout-protected/knowledgebase.php` we get the contents of the wiki. Scrolling through the lorem ipsum text, we can find `<b>MetaCTF Flag: rest_in_peace_php_die_function</b>`. 

## Lab Equipment Reservation
![lab_equipment_1.png](/assets/images/metactf/lab_equipment_1.png)
This problem starts off looking similar to the previous. Based on the URL (https://problems.metactf.com/content/sq-labs-nb) this is probably a SQL injection challenge. I tried the login page first but the username and password fields appeared to not be injectable. There is however another page, and it has a search box.
![lab_equipment_2.png](/assets/images/metactf/lab_equipment_2.png)
I first tried adding a `'` to the search, and we get a SQLite3 error:
```sql
SQLite3 error: cannot prepare statement, unrecognized token: "'"
SELECT * FROM equipment WHERE item_name LIKE '%'%'
```
This gives us the entire query. We can easily escape this by using an apostrophe. Now it's time to find the users table. Through a small amount of trial and error, you can find that the users table is called `users`. I tried to select from this table using `'; SELECT * FROM users;` but it appears that the page is only giving us results from the first query. So we can instead append the user data to the lab data using a `UNION` operator. I tried searching with `' UNION SELECT * FROM users;` but it gives the error:
```sql
SQLite3 error: cannot prepare statement, SELECTs to the left and right of UNION do not have the same number of result columns
SELECT * FROM equipment WHERE item_name LIKE '%' UNION SELECT * FROM users;%'
```
If you use a `UNION` operator, the tables you concatenate must both have the same number of columns. To get around this, we can pad the users table with a `null` column. Running `' UNION SELECT *, null FROM users;` gives us an appropriate output, meaning the users table must have 4 columns since we know the equipment table has 5. Now the table has more rows, and a few stick out as possible usernames like the row below:
![lab_equipment_3.png](/assets/images/metactf/lab_equipment_3.png)
I noticed that the serial number column looks more like a hash for the user rows, so maybe this is the password hash? We need some way of reordering the columns so the password column is not masked. Using more trial and error, I found that the username column is named username, but the password column is not called anything normal (password, passwd, pass, pw, pwd, etc). So how do we reorder columns in SQL if we don't know their names? I tried getting the schema for the various SQLite methods for this like `pragma table_info(users)` and `.schema` but these did not process correctly or were ignored by the server. This means we need to find a way of reordering the columns without knowing their names. Using a SQL subquery, we can construct our own table and change the order of the columns. The command I used is:
```sql
UNION SELECT T.a, T.b, T.c, T.d, null FROM (SELECT "a", "b", "c", "d" UNION SELECT * FROM USERS) AS T;
```
Inside the subquery above, I create a data set with `SELECT "a", "b", "c", "d"`. SQLite by default will use these values as the names of the columns in this temporary table. Using `UNION SELECT * FROM USERS`, we now have the USERS table but with columns named a, b, c, and d. Naming this temporary table `T`, We can select the individual columns from T using `T.a`, `T.b`, etc. Looking at the rows in the users column, we can see that the columns are probably id, username, name, and password. Updating our query with more friendly column names, we have:
```sql
UNION SELECT null, T.username, T.password, null, null FROM (SELECT "id", "username", "name", "password" UNION SELECT * FROM USERS) AS T;
```
This query above reorders the columns so that the username and password columns are not masked. Now it looks like we have password hashes!
![lab_equipment_4.png](/assets/images/metactf/lab_equipment_4.png)
Unfortunately, these hashes are only 21 characters long and I'm not familiar with any hashes of that length. This column must truncate to 21 characters. Luckily, SQL has a `SUBSTR` function to return a substring. By running two different SQL queries, we can get the first and last parts of the password hash:
```sql
UNION SELECT null, T.username, SUBSTR(T.password, 0, 22), null, null FROM (SELECT "id", "username", "name", "password" UNION SELECT * FROM USERS) AS T;

UNION SELECT null, T.username, SUBSTR(T.password, 22, 22), null, null FROM (SELECT "id", "username", "name", "password" UNION SELECT * FROM USERS) AS T;
```
Running these two commands separately and then concatenating the two values from the user "bison", we get a hash of `d6be6322d62ab5222f4894049e4408bf`. This has a length of 32 characters, which looks suspiciously like MD5. Searching this hash on CrackStation or HashKiller, we find the password is `monkey12`. Logging in with username `bison`, password `monkey12`, we are presented with the flag `jimmy_needs_some_help_lets_hire_little_bobby_tables`
![lab_equipment_5.png](/assets/images/metactf/lab_equipment_5.png)