# harder

## Enumeration

Start with an `nmap` scan per usual to see which ports are open, and what services run on the respective ports. `sudo nmap -Pn -v -A 10.10.222.15`

![nmap](images/nmap.png)

The website at port `80` seems empty, so we use `gobuster` to find hidden directories. This also seems to fail, because all requests are redirected to a standard error site.

By using Burp Suite we observe a _domain_ tag in the response from the website.

```
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Sun, 20 Sep 2020 18:13:29 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Vary: Accept-Encoding
X-Powered-By: PHP/7.3.19
Set-Cookie: TestCookie=just+a+test+cookie; expires=Sun, 20-Sep-2020 19:13:29 GMT; Max-Age=3600; path=/; domain=pwd.harder.local;secure
```

This response sets a cookie that is associated with the domain `pwd.harder.local`. When sending a request through burp to _pwd.harder.local_ we see that the site now has a lot more content.

![new request](images/alterRequest.png)

We thus need to associate the IP to the webserver with _pwd.harder.local_. `sudo nano /etc/hosts` and adding `<IP_to_server> pwd.harder.local` under the IPv4 section. This means that going to the website `pwd.harder.local` will take you to the associated IP.

Going to `pwd.harder.local` prompts us with a login. 

![login](images/login.png)

Using Hydra we try to crack the password `hydra pwd.harder.local -l admin -P /usr/share/wordlists/rockyou.txt http-post-form "/index.php:action=set_login&user=^USER^&pass=^PASS^:Invalid"`, and end up getting `[80][http-post-form] host: pwd.harder.local   login: admin   password: admin`.

username: admin

password: admin