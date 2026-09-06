## **Prologue: What is OS command injection?**

OS command injection is also known as shell injection. It allows an attacker to execute operating system (OS) commands on the server that is running an application, and typically fully compromise the application and its data.

## OS command injection, simple case

<img width="1780" height="937" alt="image" src="https://github.com/user-attachments/assets/10cd4207-8efc-453b-9c82-a0364dcc272b" />

Click on an item.

<img width="1915" height="1129" alt="image" src="https://github.com/user-attachments/assets/a2b2069d-35d3-4853-90c3-36dbd9ad01bb" />

We can notice that the URL has something like `product?productId=1`. That is fishy, but let’s keep going for now with “Check stock”.

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/f6d8b2ae-5dd7-4005-bdf0-84392e8c607f" />

Note: The output, 62, is the number of the stock upon checking.

The URL remains unchanged, but upon catching the request in BurpSuite, we can see the POST request body (the data sent to the stock-check checkpoint) - `productId=1&storeId=1`

In this example, a shopping application lets the user view whether an item is in stock in a particular store. This information is accessed via a URL

`https://insecure-website.com/stockStatus?productID=381&storeID=29`

To provide the stock information, the application must query various legacy systems. For historical reasons, the functionality is implemented by calling out to a shell command with the product and store IDs as arguments:

`stockreport.pl 381 29`

This command outputs the stock status for the specified item, which is returned to the user.

The application implements no defenses against OS command injection, so an attacker can submit the following input to execute an arbitrary command:

`& echo aiwefwlguh &`

If this input is submitted in the `productID` parameter, the command executed by the application is:

`stockreport.pl & echo aiwefwlguh & 29`

The `echo` command causes the supplied string to be echoed in the output. This is a useful way to test for some types of OS command injection. The `&` character is a shell command separator. In this example, it causes three separate commands to execute, one after another. The output returned to the user is:

```
1. Error - productID was not provided 
2. aiwefwlguh
3. 29: command not found
```

The three lines of output demonstrate that:

- The original `stockreport.pl` command was executed without its expected arguments, and so returned an error message.
- The injected `echo` command was executed, and the supplied string was echoed in the output.
- The original argument `29` was executed as a command, which caused an error.

Placing the additional command separator `&` after the injected command is useful because it separates the injected command from whatever follows the injection point. This reduces the chance that what follows will prevent the injected command from executing.

That said, let’s try to modify the request to execute whoami with productId=1&storeId=1&whoami . My speculation is that it will run the command stockreport.pl (with productId and storeId as the two parameters), and the command whoami . However, this does not work. Why?

> [!NOTE]
> **Metacharacters**
>
> **Semicolon (`;`)**: Acts as a sequential command separator in Unix shells and PowerShell. It tells the shell to finish the first command and immediately start the next one.
> 
> **Pipe (`|`)**: Takes the standard output of the first command and sends it as standard input to the second command. Both commands execute even if the first one doesn't output anything useful. Works both on Windows and Unix.
> 
> **Ampersand (`&`)**: In a Linux `bash` shell, a single `&` runs the first command in the background. In Windows `cmd.exe`, a single `&` acts as a sequential separator (like `;` in Linux). However, a single `&` has a special meaning in URLs (`&` separates query parameters in HTTP requests, such as `?name=annike&age=18`). If you do not URL-encode it as `%26`, the web server might interpret it as a parameter separator for the *first* request rather than part of the injected payload.
>
> **Logical OR (`||`):** This runs the second command **only if** the first command fails (returns a non-zero exit code). If the application's native command succeeds, your injected command will not execute.
>
> **Logical AND (`&&`):** This runs the second command **only if** the first command succeeds (returns a zero exit code). If the application's native command contains an error or fails, your payload is skipped.
> 
> **Miscellany**: `0x0a` or `\n` . On Unix-based systems, you can also use backticks or the dollar character to perform inline execution of an injected command within the original command like ` injected command ` or $ (injected command).
> The different shell metacharacters have subtly different behaviors that might change whether they work in certain situations. This could impact whether they allow in-band retrieval of command output or are useful only for blind exploitation. Sometimes, the input that you control appears within quotation marks in the original command. In this situation, you need to terminate the quoted context (using `"` or `'`) before using suitable shell metacharacters to inject a new command.

There is a **HTTP request headers** (a.k.a header fields) determining the content type:

`Content-Type: application/x-www-form-urlencoded` 

The program reads it is the **HTTP form parser**, not the shell. The body is actually parsed as three fields:

```
productId = "1"
storeId   = "1"
whoami    = ""
```
The stock-check application only uses `productId` and `storeId` . `whoami` in this context is not a command, but rather a field due to the ampersand`&` . Therefore, `whoami` never becomes part of `storeId` and never reaches the shell. By contrast, `%26` , `;` and `|` stayed inside the `storeId` value, so the vulnerable code passed them onward.

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/d0d85fd4-34e3-4f07-b431-1d3745f86dee" />

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/73658528-640a-40a7-b66f-ca0050c9708f" />

Note: The pipe `|` sends the output of the first command to the second command, so “62” is not displayed if using this seperator. Also, we can also play around and change `whoami` to `ls`, `pwd`,…

This vulnerability stems from that the value `storeId` is passed directly from user to the server, which does not sanitize the inputs, including such malicious metacharacters.

## **Blind OS command injection with time delays**

```
Many instances of OS command injection are blind vulnerabilities. This means that the application does not return the output from the command within its HTTP response. Blind vulnerabilities can still be exploited, but different techniques are required.

As an example, imagine a website that lets users submit feedback about the site. The user enters their email address and feedback message. The server-side application then generates an email to a site administrator containing the feedback. To do this, it calls out to the `mail` program with the submitted details:

`mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com`

The output from the `mail` command (if any) is not returned in the application's responses, so using the `echo` payload won't work. In this situation, you can use a variety of other techniques to detect and exploit a vulnerability.

You can use an injected command to trigger a time delay, enabling you to confirm that the command was executed based on the time that the application takes to respond. The `ping` command is a good way to do this, because lets you specify the number of ICMP packets to send. This enables you to control the time taken for the command to run:

`& ping -c 10 127.0.0.1 &`

This command causes the application to ping its loopback network adapter for 10 seconds.
```

This challenge has nothing to do with the homepage, so let’s hop into the “Submit Feedback” and test it out.

<img width="1674" height="888" alt="image" src="https://github.com/user-attachments/assets/ef77abaf-2709-4037-9332-a15c8db82479" />

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/3c2d9ff2-9d01-4b00-9b6b-ba2fc321378f" />

> [!NOTE]
> **CSRF** means **Cross-Site Request Forgery**. The long value after `csrf=` is an **anti-CSRF token**. The server generated it for your logged-in session and expects it back when you submit the feedback form. This is irrelevant to this challenge, just a sidenote.
>
> Our target should rather be `email` in this lab because the feedback feature sends the submitted feedback by invoking a mail-related shell command, with user-supplied details included in it. The lab deliberately makes the `email` field the unsafe insertion point; its output is not returned, hence it is “blind.”

Since I have read the documentation beforehand, I will delve into the intended solution a bit.

`email=luna||ping+-c+10+127.0.0.1||` 

Since the email field require `@` (and the suffix), truncating it back to a plain “luna” renders the request to be failed. However, by implementing the metachar logical OR, it will proceed the second command, which is delaying 10 seconds, before returning the result that it “could not save”.

Why do we have to put the seperator at both the start and the end of the payload? Simply because the vulnerable application places the email value inside a mail command. Conceptually, the shell ends up seeing:

`… From: luna || ping -c 10 127.0.0.1 || Subject: …`

The second seperator separates your command from the server’s remaining text. Because ping succeeds, that trailing fragment is not run as a command.

Nevertheless, I find this approach somewhat overtly complicated. 

- First, the `ping` command can be replaced with the `sleep` command, which is more simple to me.
- Second, the intended solution use the logical OR seperator, but this is kinda overkill. I suppose there is a way to leverage our trustworthy ampersand, similar to the previous challenge.

`email=luna%40ctf.org+%26+sleep+10+%26`  or  `email=luna%40ctf.org+%26+sleep+10+%23` works.

Note: 

- `%26` is the ampersand as we know it.
- `%23` is the hash symbol `#` , used to turn what follows into comment. This also works in this scenario if put at the end. Conceptually, this technique is pretty self-explanatory.

## **Blind OS command injection with output redirection**

First and foremost, we (blindly) test if there is an OS command injection using the technique from the previous lab (time delays). Indeed, the response is again slept for 10 seconds, so there remains the same vulnerability. Change the payload from `email=luna%40ctf.org+%26+sleep+10+%26`  to:

`email=luna%40ctf.org+%26+whoami+>+/var/www/images/whoami.txt+%26` 

This serves as an output direction, and befitting to our objective this time. Rather than just sleeping the response, we need to `whoami` this time. However, the output from the command is not returned in the response (like the simple case), and we need some sort of workaround for this. 

Luckily, we can use output redirection to capture the output from the command. There is a writable folder at: `/var/www/images` .

The application serves the images for the product catalog from this location. You can redirect the output from the injected command to a file in this folder, and then use the image loading URL to retrieve the contents of the file.

<img width="1915" height="1126" alt="image" src="https://github.com/user-attachments/assets/a9f5994d-1988-4ed7-af93-97e17d7e3271" />

Upon opening HTTP history (and turn off the filtering for images), so will see a handful amount of GET method from the folder `/images` with the parameter `?filename=....jpg` . These are the images we see in the shop. Since we have copied the output of the `whoami` command into the `whoami.txt` in this folder, we just send the GET method from the `/images` to the repeater and change the parameter to `?filename=whoami.txt` to retrieve the output.

## Blind OS command injection with out-of-band interaction

## Blind OS command injection with out-of-band data exfiltration

## Epilogue: How to prevent OS command injection attacks?

The most effective way to prevent OS command injection vulnerabilities is to never call out to OS commands from application-layer code. In almost all cases, there are different ways to implement the required functionality using safer platform APIs.

If you have to call out to OS commands with user-supplied input, then you must perform strong input validation. Some examples of effective validation include:

- Validating against a whitelist of permitted values.
- Validating that the input is a number.
- Validating that the input contains only alphanumeric characters, no other syntax or whitespace.

Never attempt to sanitize input by escaping shell metacharacters. In practice, this is just too error-prone and vulnerable to being bypassed by a skilled attacker.
