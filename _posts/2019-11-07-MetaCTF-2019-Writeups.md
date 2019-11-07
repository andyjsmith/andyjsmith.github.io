---
title: MetaCTF 2019 Writeups
author: Andy Smith
---

# Web Exploitation
## C3 Internal Knowledge Base
![c3_employee_knowledgebase.png](/assets/metactf/c3_employee_knowledgebase.png)
In this problem we're given a login page and need to access the internal wiki. Trying to browse to the wiki gives a 302 Found redirect with a header `Location: ./?error=unauthorized&source=./knowledgebase.php`. Chrome happily follows this redirect and does not display any information in the response body, however if we use a command line tool like `curl` we can just ignore this redirect. Running `curl https://problems.metactf.com/content/logout-protected/knowledgebase.php` we get the contents of the wiki. Scrolling through the lorem ipsum text, we can find `<b>MetaCTF Flag: rest_in_peace_php_die_function</b>`. 