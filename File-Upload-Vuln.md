## What are file upload vulnerabilities:

File upload vulnerabilities are when a web server allows users to upload files to its filesystem without sufficiently validating things like their name, type, contents, or size. Failing to properly enforce restrictions on these could mean that even a basic image upload function can be used to upload arbitrary and potentially dangerous files instead. This could even include server-side script files that enable remote code execution.

In some cases, the act of uploading the file is in itself enough to cause damage. Other attacks may involve a follow-up HTTP request for the file, typically to trigger its execution by the server.

--- 

## What is the impact of file upload vulnerability:

The impact of file upload vulnerabilities generally depends on two key factors:

- Which aspect of the file the website fails to validate properly, whether that be its size, type, contents, and so on.
- What restrictions are imposed on the file once it has been successfully uploaded.

In the worst case scenario, the file's type isn't validated properly, and the server configuration allows certain types of file (such as `.php` and `.jsp`) to be executed as code. In this case, an attacker could potentially upload a server-side code file that functions as a web shell, effectively granting them full control over the server.

If the filename isn't validated properly, this could allow an attacker to overwrite critical files simply by uploading a file with the same name. If the server is also vulnerable to directory traversal, this could mean attackers are even able to upload files to unanticipated locations.

Failing to make sure that the size of the file falls within expected thresholds could also enable a form of denial-of-service (DoS) attack, whereby the attacker fills the available disk space.

---

## How do file upload vulnerability arise:

Given the fairly obvious dangers, it's rare for websites in the wild to have no restrictions whatsoever on which files users are allowed to upload. More commonly, developers implement what they believe to be robust validation that is either inherently flawed or can be easily bypassed.

For example, they may attempt to blacklist dangerous file types, but fail to account for parsing discrepancies when checking the file extensions. As with any blacklist, it's also easy to accidentally omit more obscure file types that may still be dangerous.

In other cases, the website may attempt to check the file type by verifying properties that can be easily manipulated by an attacker using tools like Burp Proxy or Repeater or caido replay and proxy.

Ultimately, even robust validation measures may be applied inconsistently across the network of hosts and directories that form the website, resulting in discrepancies that can be exploited.

--- 

## How web servers handle requests for static file:

The process for handling these static files is still largely the same. At some point, the server parses the path in the request to identify the file extension. It then uses this to determine the type of the file being requested, typically by comparing it to a list of preconfigured mappings between extensions and MIME types. What happens next depends on the file type and the server's configuration.

- If this file type is non-executable, such as an image or a static HTML page, the server may just send the file's contents to the client in an HTTP response.
- If the file type is executable, such as a PHP file, **and** the server is configured to execute files of this type, it will assign variables based on the headers and parameters in the HTTP request before running the script. The resulting output may then be sent to the client in an HTTP response.
- If the file type is executable, but the server **is not** configured to execute files of this type, it will generally respond with an error. However, in some cases, the contents of the file may still be served to the client as plain text. Such misconfigurations can occasionally be exploited to leak source code and other sensitive information. You can see an example of this in our information disclosure learning materials.# Starting phase

Before we look at how to exploit file upload vulnerabilities, it's important that you have a basic understanding of how servers handle requests for static files.

Historically, websites consisted almost entirely of static files that would be served to users when requested. As a result, the path of each request could be mapped 1:1 with the hierarchy of directories and files on the server's filesystem. Nowadays, websites are increasingly dynamic and the path of a request often has no direct relationship to the filesystem at all. Nevertheless, web servers still deal with requests for some static files, including stylesheets, images, and so on.

The process for handling these static files is still largely the same. At some point, the server parses the path in the request to identify the file extension. It then uses this to determine the type of the file being requested, typically by comparing it to a list of preconfigured mappings between extensions and MIME types. What happens next depends on the file type and the server's configuration.

- If this file type is non-executable, such as an image or a static HTML page, the server may just send the file's contents to the client in an HTTP response.
- If the file type is executable, such as a PHP file, **and** the server is configured to execute files of this type, it will assign variables based on the headers and parameters in the HTTP request before running the script. The resulting output may then be sent to the client in an HTTP response.
- If the file type is executable, but the server **is not** configured to execute files of this type, it will generally respond with an error. However, in some cases, the contents of the file may still be served to the client as plain text. Such misconfigurations can occasionally be exploited to leak source code and other sensitive information. You can see an example of this in our information disclosure learning materials.
## TIP: 

The `Content-Type` response header may provide clues as to what kind of file the server thinks it has served. If this header hasn't been explicitly set by the application code, it normally contains the result of the file extension/MIME type mapping.

--- 

## Exploiting unrestricted file upload vulnerability:

From a security perspective, the worst possible scenario is when a website allows you to upload server-side scripts, such as PHP, Java, or Python files, and is also configured to execute them as code. This makes it trivial to create your own web shell on the server.

#### Web shell

A web shell is a malicious script that enables an attacker to execute arbitrary commands on a remote web server simply by sending HTTP requests to the right endpoint.

If you're able to successfully upload a web shell, you effectively have full control over the server. This means you can read and write arbitrary files, exfiltrate sensitive data, even use the server to pivot attacks against both internal infrastructure and other servers outside the network. For example, the following PHP one-liner could be used to read arbitrary files from the server's filesystem:

`<?php echo file_get_contents('/path/to/target/file'); ?>`

Once uploaded, sending a request for this malicious file will return the target file's contents in the response.

A more versatile web shell may look something like this:

`<?php echo system($_GET['command']); ?>`

This script enables you to pass an arbitrary system command via a query parameter as follows:

`GET /example/exploit.php?command=id HTTP/1.1`

---

## One simple process: 

1. While proxying traffic through Burp or caido, log in to your account and notice the option for uploading an avatar image.
2. Upload an arbitrary image, then return to your account page. Notice that a preview of your avatar is now displayed on the page.
3. In Burp or caido, go to **Proxy > HTTP history**. Click the filter bar to open the **HTTP history filter** window. Under **Filter by MIME type**, enable the **Images** checkbox, then apply your changes. (By default in caido that image is also enabled)
4. In the proxy history, notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater or caido replay.
5. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret file. For example:
    
    `<?php echo file_get_contents('/etc/passwd'); ?>`
6. Use the avatar upload function to upload your malicious PHP file. The message in the response confirms that this was uploaded successfully.
7. In Burp Repeater or caido replay, change the path of the request to point to your PHP file:
    
    `GET /files/avatars/exploit.php HTTP/1.1`
8. Send the request. Notice that the server has executed your script and returned its output (command output) in the response.

--- 

## Exploiting flawed validation of file uploads

In the wild, it's unlikely that you'll find a website that has no protection against file upload attacks like we saw in the previous technique. But just because defenses are in place, that doesn't mean that they're robust. You can sometimes still exploit flaws in these mechanisms to obtain a web shell for remote code execution.

--- 

## Flawed file type validation

When submitting HTML forms, the browser typically sends the provided data in a `POST` request with the content type `application/x-www-form-urlencoded`. This is fine for sending simple text like your name or address. However, it isn't suitable for sending large amounts of binary data, such as an entire image file or a PDF document. In this case, the content type `multipart/form-data` is preferred.

--- 

## Flawed file type validation - Continued

Consider a form containing fields for uploading an image, providing a description of it, and entering your username. Submitting such a form might result in a request that looks something like this:

```POST /images HTTP/1.1 Host: normal-website.com Content-Length: 12345 Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456 ---------------------------012345678901234567890123456 Content-Disposition: form-data; name="image"; filename="example.jpg" Content-Type: image/jpeg [...binary content of example.jpg...] ---------------------------012345678901234567890123456 Content-Disposition: form-data; name="description" This is an interesting description of my image. ---------------------------012345678901234567890123456 Content-Disposition: form-data; name="username" wiener ---------------------------012345678901234567890123456--
```

As you can see, the message body is split into separate parts for each of the form's inputs. Each part contains a `Content-Disposition` header, which provides some basic information about the input field it relates to. These individual parts may also contain their own `Content-Type` header, which tells the server the MIME type of the data that was submitted using this input.

---

## Flawed file type validation - Continued

One way that websites may attempt to validate file uploads is to check that this input-specific `Content-Type` header matches an expected MIME type. If the server is only expecting image files, for example, it may only allow types like `image/jpeg` and `image/png`. Problems can arise when the value of this header is implicitly trusted by the server. If no further validation is performed to check whether the contents of the file actually match the supposed MIME type, this defense can be easily bypassed using tools like Burp Repeater or caido replay.

---

## Process Steps:

- Log in and upload an image as your avatar, then go back to your account page.
- In Burp or caido, go to **Proxy > HTTP history** and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater.
- On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret. For example:
    
    `<?php echo file_get_contents('/etc/passwd'); ?>`
- Attempt to upload this script as your avatar. The response indicates that you are only allowed to upload files with the MIME type `image/jpeg` or `image/png`.
- In Burp or caido, go back to the proxy history and find the `POST /my-account/avatar` request that was used to submit the file upload. Send this to Burp Repeater.
- In Burp Repeater or caido replay, go to the tab containing the `POST /my-account/avatar` request. In the part of the message body related to your file, change the specified `Content-Type` to `image/jpeg`.
- Send the request. Observe that the response indicates that your file was successfully uploaded.
- Switch to the other Repeater tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that command's output was returned in the response.

--- 

> T0 be continued.............. (^_^)
