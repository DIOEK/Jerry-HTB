# Jerry-HTB
```bash sudo nmap -T4 --min-rate 1000 -A -sC -p8080 jerry.htb
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-10 15:07 -0300
Nmap scan report for jerry.htb (10.129.136.9)
Host is up (0.080s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache Tomcat/7.0.88
|_http-server-header: Apache-Coyote/1.1
|_http-favicon: Apache Tomcat
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2012|2008|7 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7
Aggressive OS guesses: Microsoft Windows Server 2012 R2 (97%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2012 or Windows Server 2012 R2 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

TRACEROUTE (using port 8080/tcp)
HOP RTT      ADDRESS
1   83.90 ms 10.10.16.1
2   84.01 ms jerry.htb (10.129.136.9)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.97 seconds
```

At port 8080, we have a instance of apache tomcat a webserver proxy from apache. Let's fuzz it:
```bash
ffuf -w ../../../../SecLists/Discovery/Web-Content/raft-large-directories.txt -u http://jerry.htb:8080/FUZZ


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://jerry.htb:8080/FUZZ
 :: Wordlist         : FUZZ: /home/spaz/SecLists/Discovery/Web-Content/raft-large-directories.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

docs                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 198ms]
manager                 [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 306ms]
examples                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 51ms]
host-manager            [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 55ms]
:: Progress: [62281/62281] :: Job [1/1] :: 701 req/sec :: Duration: [0:01:38] :: Errors: 0 ::
```
From these manager is interesting, if we try to access it it asks for credentials that can bee faound in googlem thay are (tomcat/s3cret):
<img width="1919" height="921" alt="image" src="https://github.com/user-attachments/assets/87cad964-9bfc-4cb1-a4b0-e8e4bec01401" />

Inside the tomcat manager page we can find this:
<img width="1919" height="801" alt="image" src="https://github.com/user-attachments/assets/cb5e3ac4-ce9a-498a-8666-c715cbc9046c" />
