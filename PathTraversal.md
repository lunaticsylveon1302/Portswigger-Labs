## What is Path Traversal?
Path traversal is also known as directory traversal. These vulnerabilities enable an attacker to read arbitrary files on the server that is running an application.

## File path traversal, simple case
```
This lab contains a path traversal vulnerability in the display of product images. 
To solve the lab, retrieve the contents of the /etc/passwd file.
```

First, change the filter so that it display the product images.

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/fe426562-5787-49c9-a3ed-8829ec2efa97" />

Send it to Repeater and modify the request to `../../../etc/passwd` .

<img width="1918" height="1123" alt="image" src="https://github.com/user-attachments/assets/dfb12cc5-d45b-4ed1-b101-ca3c8037a3c0" />

`../` is a path traversal sequence, moving up the filesystem by one folder. 

Why we have to move up the filesystem three times (or more), you may ask? Web servers typically store website files deep inside a specific folder called the **web root**. A common path on Linux for a website file is `/var/www/images/photo.jpg` .

> [!TIP]
> To return an image, the application appends the requested filename to this base directory and uses a filesystem API to read the contents of the file. In other words, the application reads from the following file path: `/var/www/images/218.png` .
>
> This application implements no defenses against path traversal attacks. As a result, an attacker can request the following URL to retrieve the `/etc/passwd` file from the server's filesystem. This causes the application to read from the following file path:
>
> `/var/www/images/../../../etc/passwd`
>
> The sequence `../` is valid within a file path, and means to step up one level in the directory structure. The three consecutive `../` sequences step up from `/var/www/images/` to the filesystem root, and so the file that is actually read is:
>
> `/etc/passwd`

## **File path traversal, traversal sequences blocked with absolute path bypass**
```
The application blocks traversal sequences but treats the supplied filename as being relative to a default working directory.
```
Many applications that place user input into file paths implement defenses against path traversal attacks, which can be bypassed. If an application strips or blocks directory traversal sequences from the user-supplied filename, it might be possible to bypass the defense using a variety of techniques. 

For example, you might be able to use an absolute path from the filesystem root, such as `filename=/etc/passwd`, to directly reference a file without using any traversal sequences.

</aside>

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/e61d6ba3-d7f4-4ed1-b473-38ed4b18feb4" />

## **File path traversal, traversal sequences stripped non-recursively**
```
The application strips path traversal sequences from the user-supplied filename before using it.
```
Let’s take the exploit for the simple case lab for example: `../../../etc/passwd` . The application will strip the traversal sequences, so what is left is only `/etc/passwd` . 

Note that the stripping is non-recursive. Simply adding more sequences will not work. However, a smart workaround is `....//....//....//etc/passwd` . Since the application will strip the traversal sequences in a non-recursive way, it will delete every `../` only once. After stripping, what is left is our original exploit `../../../etc/passwd` .

## **File path traversal, traversal sequences stripped with superfluous URL-decode**

```
The application blocks input containing path traversal sequences.
It then performs a URL-decode of the input before using it.
```

This sentence is from [an OWASP article](https://owasp.org/www-community/Double_Encoding):

**By using double encoding it’s possible to bypass security filters that only decode user input once. The second decoding process is executed by the backend platform or modules that properly handle encoded data, but don’t have the corresponding security checks in place.**

It is noteworthy that, the description is kinda deceptive here. The true workflow is that, the application will first decode a URL-decode of our payload, then check if it contains any path traversal sequences. If not, the backend platform will handle the encoded data (by performing another URL-decode), thereby reconstructing our path traversal payload.

Let’s say that our exploit remains to be `../../../etc/passwd` . First we encode it one time to `..%2F..%2F..%2Fetc%2Fpasswd` . However, this will fail the security check as it will be decoded back to the path traversal sequence and will be blocked. We will then double encode it to `..%252F..%252F..%252Fetc%252Fpasswd` . This will not return the path traversal sequence immediately upon decoding once, thereby passing the security check.

Various non-standard encodings, such as `..%c0%af` or `..%ef%bc%8f`, may also work.

<img width="1918" height="1126" alt="image" src="https://github.com/user-attachments/assets/b9ddfa7e-0b93-4d2a-9f73-f49cf90f77a1" />

## File path traversal, validation of start of path

```
The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.
```

The solution is similar to my callout in the simple case.

<img width="1918" height="1125" alt="image" src="https://github.com/user-attachments/assets/10e7bf69-a066-4033-93e9-ba136d736565" />

## File path traversal, validation of file extension with null byte bypass

```
The application validates that the supplied filename ends with the expected file extension.
```

An application may require the user-supplied filename to end with an expected file extension, such as `.png`. In this case, it might be possible to use a null byte to effectively terminate the file path before the required extension. 

For example: `filename=../../../etc/passwd%00.png`

A null byte is a control character with a value of zero (`\0` in code, `0x00` in hex, or `%00` in URLs) used to mark the end of a string in low-level programming languages like C and C++.

A null byte injection in path traversal is a technique used to bypass file extension checks by terminating the string early. The defense mechanism checks user input to ensure a file path ends with a specific extension such as `.jpg` or `.png` . The application logic handles the full string with the extension, but the underlying system or C-based file API treats a null byte (`\0` or URL-encoded `%00`) as the end of the string. 

Therefore, an attacker can input a path traversal sequence followed by a null byte and the required extension (e.g., `../../../etc/passwd%00.png`). The validator sees `.png` at the end and allows the request, but the file system stops processing at `%00` and opens `/etc/passwd` directly.

## Epilogue: How to prevent a path traversal attack?

The most effective way to prevent path traversal vulnerabilities is to avoid passing user-supplied input to filesystem APIs altogether. Many application functions that do this can be rewritten to deliver the same behavior in a safer way.

If you can't avoid passing user-supplied input to filesystem APIs, we recommend using two layers of defense to prevent attacks:

- Validate the user input before processing it. Ideally, compare the user input with a whitelist of permitted values. If that isn't possible, verify that the input contains only permitted content, such as alphanumeric characters only.
- After validating the supplied input, append the input to the base directory and use a platform filesystem API to canonicalize the path. Verify that the canonicalized path starts with the expected base directory.

Below is an example of some simple Java code to validate the canonical path of a file based on user input:
```
File file = new File(BASE_DIRECTORY, userInput);
if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
    // process file
}
```
