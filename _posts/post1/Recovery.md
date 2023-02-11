---
title: "TryHackMe WriteUp: Recovery"
date: 2022-09-12T07:10:15+05:30
---


Hey there, Now a days i was solving THM and HTB machines and thought  lets write
how i solved and where i failed and what i learned, i wil be writing on machines who i think i learned something new and different from others

This time i am writing on Recovery machine , this is medium  level room if you haven't try it yet i will not force you  but hey you should try first and if you are here just because you tried it and got stuck then just solve it together :) ya muchh talking :D 






![alt text](../imgs/machine-logo.png)  







Note: Read the msg by alex from challenge description what exactly he is saying it will help you to recovery his system as he is the one who know better what exactly happend to him  .


So as always first search for open ports and services running on this machine with nmap 


![nmap result](../imgs/nmap.png)

and look at here ohmydash what i got :0 http and ssh are running simultaniously 
that i already knew it as we have been told we can check our flags at http://$machine_ip:1337 and ssh login credentials to login in to alex's system is 


``username: alex
``password: madeline``


After ssh ing a message was printing in a loop

![crazy msg](../imgs/msg.png)

## Hunting Flag0 

### Aproach #1 to get first flag

Look like there is some script progrmmed or job shedule which run after booting the system  which is possilbe through .bashrc file or crontab  i looked for  .bashrc file and there was line written at the end of the file for looping that message


![bashrc](../imgs/bashrcfile.png)

> ?: how did i stop that msg while loop :  justing by hitting 5-10 times Enter and Ctrl^c :D thats how it worked for me give a try it's fun :P .  actually i figurd it out later reading someone writeup i could just simply copy the my local machine .bashrc file with alex.


after deleting the the line i got my first flag (you can che)


![first flag](../imgs/flag0.png)


### Aproach #2 to get first flag

In home dir there was binary file **fixutil** and this is the file alex was talking about for better convenient i copied on my local system with **scp** utility .

run on local system

```command
scp alex@$IP:/home/alex/fixutil local/dir/
```

i looked for strings in binary to get idea about what this binary up to through **strings** utility i had an idea  what's going on 


![stirngs](../imgs/strings.png)


let's open this binary in ghidra and decompile it into pseudocode and look into main() function 'cause it's main function


![fixutil](../imgs/fixutil.png)


pretty clean huh? , yeah in above image we can see that there is FILE variable **pFvar1** which containing the **.bashrc** file and writing some shell script into that var(.bashrc file) and 'cause it's .bashrc file we know that **bashrc file is a script file that's executed when a user logs in** this ended up with looping message when i loggin into the system.
remove it from the file and this way i get the first flag . 



## Hunting Flag1

meanwhile i faced issue that my loggin session is disconnecting automatially firstly thought that it was from my side but then remeber fix the fixutil strings i have seen earlier that there is job sheduled in crontab look at this image below .

![crontab](../imgs/cronstring.png)

/opt/ dir listing 


![opt dir](../imgs/opt_dir.png)


there is one dir **.fixutil** which has permission and as i am local user for now can't access this dir and  a script **brilliant_script.sh** in **/opt/** directory wich is sheduled to run every minute 
what we can figure out from this script that there **for loop** running on to get the process of bash shell using grep command and taking it's PID to KILL with **kill** command so just overwrite it and now no more session breakation issue :D .

but wait a minute did you  notice the file is a **.sh** file and with job sheduled in crontab as a root user it means i can get escalate our priviledge from alex to root 

here i use bash as python wasn't there for reverse shell you can use [revshell](https://www.revshells.com/)
for all type of reverse shell so writing bash revshell into brilliant_script.sh file 

![revshell](../imgs/revshell.png)

and 

![meme](../imgs/eternity.jpg)


I AM GROOT! . 

![rootshell](../imgs/rootshell.png)


and captured the secod flag too. 


![flag2](../imgs/flag1.png)


lets see the whats inside that **.fixutil** directory 


![bakcup](../imgs/backuptxt.png)

hmm, some gibberish strings whas the use of this strings for that time i left as is as i didn't have idea what to do with this file .

>Note: actually this room has total 6 flag and in my case while solving this room i captured the flags non sequentially and i am going to write as is , as  it will make more sense like a puzzle you get a random hint and trying to find flag relating to that hint at the last what it call CTF. 



## Hunting Flag3

0kay, until now i have 2 flags and 4 remaining. 
retrace the path and see what footprint i have (actually there is lot assuming just by the result of that fixutil binary strings resutl) .
hmm , fixutil reminded me that there was something too 


![fixutil2](../imgs/fixutil2.png)


at 10th line there is system() function copying the **liblogging.so** file from /lib/x86_64-linux-gnu/ to /tmp/ dir and fwrite() function writing that file and lastly there is another system() func echoing string 'pwned' and passing to a file through pipe 

here i have two file to look up 

1. liblogging.so
2. admin 

copying the liblogging.so file to local directory using scp utility like i did for previous file and decomping it using ghidra lets see what i get 
there are 2 interesting function 

1 logIncorrectAttempt
2 XorEncryptWebFile
	-  XORFile 


1. logIncorrectAttempt


![liblogging](../imgs/logincorrectattempt.png)


First Arrow :  moving the /tmp/logging.so fie to /lib/x86_64-linux-gnu/ as oldliblogging.so this is the previous file we have copied from /lib/x86_64-linux-gnu/liblogging to /tmp/ as a logging.so 

Second Arrow : opening the authorized_keys file from /root/.ssh/ dir as writable permission and storing in pFVar2 var

Curly Braces : writing public key to that pFVar2 var 

Third Arrow and square bracket : adding a new use 'secuirty' with passwd

Last two arrow : we have figured it out already 

0kay , so  jumping to the Second Arrow as in first i had to go through that file 
just opening the authorized_keys file and writing the public key look like attacker made it to escalate priviledge and as i have to recover the system i overwrited the  authorized_keys file with empty data/deleted the public key from inside the file or can delete directly  and captured the flag3


![flag3](../imgs/flag3.png)



## Hunting Flag4

Move to Third Arrow and square bracket attacker adding new user 'security' and pasword as well, removing this user from system using **userdel** command or can do it manually (removing entries from /etc/passwd and /etc/shadow) 
and doing this  i got my flag4 

![flag4](../imgs/flag4.png)


Left with last two flag i.e flag2 and flag5 



## Hunting Flag2


First Arrow :  moving the /tmp/logging.so fie to /lib/x86_64-linux-gnu/ as oldliblogging.so this is the previous file we have copied from /lib/x86_64-linux-gnu/liblogging to /tmp/ as a logging.so let just undo it to its realname .

`cd /lib/x86_64-linux-gnu/ && mv /lib/x86_64-linux-gnu/oldliblogging liblogging.so 

and doing this  i got my flag2

![](../imgs/flag2.png)



## Hunting Flag5


2. XorEncryptWebFile


![xorencwebfile](../imgs/xorexplaining.png)


After reviewing the ghidra gernerated pseudo code what i get to know that this function this function generating the encryption key and storing that key into dir /opt/.fixutil/ as backup.txt and if dir doesn't exist creating the new one and as i have already had cup of tea with this text file it does contain encryption key 

curly braces 2: function 'GetWebFiles' taking the all file from /usr/local/apache2/htdocs/ 


![getwebfiles](../imgs/weblocation.png)


one by one and encrypting it with the xoring with encryption key


![xoring](../imgs/xoring.png)


so i copied all all encrypted file from /usr/local/apache2/htdocs/ to alex home dir as i require root permission and from then copied to my local machine using **scp** command 
and decrypted all files with encryption key by reversing the encryption process using this python script


```python

#!/usr/bin/env python 
from itertools import cycle

key = "AdsipPewFlfkmll"
#fname = "index."
data  = open("./htdoc/reallyimportant.txt", "rb").read()
xor = [chr(ord(a) ^ ord(b)) for a,b in zip(data, cycle(key))]
content = ''.join(map(str, xor))
print content
wome = open("./decrypted/reallyimportant.txt", "w") 
wome.write(content) 

```



after decrypting the each file with same key 


![exp result](../imgs/decryptedmsg.png)



as i have been undoing the all changes made by attacker i had to replace all files with decrypted one so.
i copied all files back to its dir and as it require root permission and althogh we have root shell but pass so copied to into alex home dir and then its residential dir from root shell
and i got my last flag with last enter .


![flag5](../imgs/flag5.png)



2.  admin


![adminbin](../imgs/adminbin.png)


still left with this binary  and nothing usefull, decompiling with ghidra we can see there is passphrase hardcoded using that we end up with msg "This section is currently under development, sorry" i was just rabbit hole .



## Summary

Overall challenges in this room was quite good author designed room as real case scenerio compare to all othere room i have played so far as metioned not convetional CTF recovery all compromised system tracking all footprint left by attacker via reversing , analysing malware. 
