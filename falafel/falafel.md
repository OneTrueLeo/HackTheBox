Initial Recon

Nmap

![image](https://user-images.githubusercontent.com/88967140/178944865-a3ef8183-1cd8-4fed-ae77-66f69aceacd8.png)

Web Directory Enumeration (Gobuster)
![image](https://user-images.githubusercontent.com/88967140/178944919-596a497a-5a90-444c-9b73-c7029cb28908.png)
![image](https://user-images.githubusercontent.com/88967140/178944927-a6a217c7-5192-4961-bd8d-f18cba0dd4a2.png)

* cyberlaw.txt looked like an interesting text page, which disclosed a user named `chris`.
* ![image](https://user-images.githubusercontent.com/88967140/178945084-83ed301e-ba19-4c43-ada6-9f99fe908af5.png)

Initial Foothold

SQL Injection

Since there wasn't anything else other than the login form, I captured the login POST via Burp and used it in `sqlmap`.
![image](https://user-images.githubusercontent.com/88967140/178946925-b0d2ebe8-d070-4991-92ed-030cef796d44.png)
![image](https://user-images.githubusercontent.com/88967140/178946947-2a01e5a4-efca-491f-b15f-a59adc17788d.png)

SQLMap

Sending different usernames in the login form returned different results - 
![image](https://user-images.githubusercontent.com/88967140/178949194-20b06a3f-c7de-47b4-91ab-7ad42d387bf1.png)
![image](https://user-images.githubusercontent.com/88967140/178949227-3d73042b-ec92-49e0-9b8a-2ce02c366be2.png)
![image](https://user-images.githubusercontent.com/88967140/178949211-e65cf870-43d0-4f3a-a36a-a08d21bc77ab.png)

Notice how entering `admin` as the username would show `Wrong identification` in the response, but using `test` would show `Try again..`.

By this it can be deduced that this login form is vulnerable to a boolean-based blind injection (the username= parameter).

So basically "Wrong identification" = TRUE, "Try again.." = FALSE.
