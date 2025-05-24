---
layout: default
---
[Volver al inicio](https://alejandromartinezmoreno.github.io/CyberHustler)
# Máquina Virtual: Lame
**Plataforma:** Hack The Box
**Categoría:** Uso de exploits con Metasploit

---

## Información General
- **IP de la máquina:** `10.10.10.245`
- **Objetivo:** Obtener dos flags: user y root
- **Herramientas utilizadas:** Nmap, Curl, TCPdump, etc.

---

## Resolución

# Reconocimiento

La IP objetivo será 10.10.10.3, mientras que la máquina atacante responderá a la IP 10.10.14.87.

# Escaneo

Del mismo modo que en Strutted, la fase de escaneo comenzará con un escaneo de puertos Nmap. Aunque, en esta ocasión, se modificará el comando usado, con el objetivo de conocer más a fondo las posibilidades de esta herramienta.

![Escaneo Nmap](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/1.nmap.jpg)

El comando -T5 tiene un efecto similar a --min-rate 5000, ambos ejecutan un escaneo veloz. Si bien en la máquina anterior se explicó que --min-rate establecía la tasa de paquetes por segundo, -T tiene un funcionamiento similar. -T va acompañado de un número del 1 al 5, siendo -T1 un escaneo sigiloso (<20 paquetes por segundo), y -T5 un escaneo agresivo (>1000 paquetes por segundo).

Por otro lado, al añadir -oA (nombre) el resultado del escaneo se guardará bajo el nombre establecido (en este caso “nmap”) en tres diferentes extensiones: .gnmap, .nmap y .xml.

Así, el escaneo da como resultado 4 puertos abiertos.

El primer puerto mostrado es el 21/TCP, en el que se encuentra en ejecución el protocolo FTP (File Transfer Protocol). FTP permite el envío y recepción de archivos entre equipos mediante una red. En este caso, el protocolo se lleva a cabo mediante el programa vsftpd en su versión 2.3.4.

Se sabe que el protocolo FTP permite accesos anónimos mediante las credenciales anonymous/anonymous. Pese a que solo se podrán ver aquellos archivos que sean públicos, es una buena práctica en la fase de escaneo y búsqueda de información.

![Login Anónimo FTP](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/2.ftp.jpg)

Como es de apreciar, no existen directorios públicos o accesibles desde el login anónimo.

El siguiente puerto abierto detectado es el 22/TCP, donde se ejecuta el protocolo SSH, al igual que en Strutted. Se observa que la versión usada de OpenSSH no es la encontrada en la máquina anterior, sino que es antigua (versión 4.7p1).

Por último, en los puertos 139/TCP y 445/TCP se está ejecutando Samba. Samba es un programa que permite la conexión entre un PC Linux o Unix y un PC Windows mediante SMB (Server Message Block), para transferencia de archivos en sistemas Windows. En términos llanos, “traduce” el idioma Linux para comunicarse con un equipo Windows. A diferencia de FTP, que es universal, SMB es nativo de Windows, por lo que, para hacer uso desde otro sistema operativo, se deberán utilizar herramientas como Samba.

La diferencia entre los puertos 139 y 445 radica en el funcionamiento del programa. En el puerto 139 se establece la conexión mediante NetBIOS, un sistema, ya anticuado, que permite a los equipos reconocerse y comunicarse mediante SMB. Por otro lado, en el puerto 445, pese a que la captura de la Figura [] puede llevar a confusión, no se está utilizando NetBIOS. La conexión SMB es directa, lo que supone una mayor eficiencia. Por tanto, se deberá interceptar el puerto en el que se esté llevando a cabo la transferencia de archivos.

En cuento a la versión de Samba utilizada, se encuentra entre todas las versiones de Samba 3 y 4, sin que se especifique. Sin embargo, el siguiente comando de nmap aporta información adicional sobre el puerto, entre la que se encuentra a versión específica. Véase la siguiente figura.

![Puerto 139](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/3.p139.jpg)

--script smb-os-discovery aporta detalles extra sobre el equipo, como el sistema operativo, el nombre del equipo y dominio y la versión del programa usado en el puerto especificado (-p139). Así, se descubre que la versión de Samba usada es la 3.0.20-Debian. Además, al ser en el puerto 139, como se comentó anteriormente, la conexión está establecida mediante NetBIOS.

Hágase hincapié en las versiones de los programas usados. La versión de vsftpd 2.3.4 ha obtenido fama dentro del mundo de la ciberseguridad por ser vulnerable. El mismo caso es la versión 4.7p1. Con el fin de explotar ambas vulnerabilidades, y explorar el protocolo Samba en busca de opciones, se presentará la herramienta Metasploit.

Metasploit es una plataforma diseñada para encontrar y explotar vulnerabilidades en sistemas informáticos de forma automatizada. Dentro de su abanico de opciones se encuentran:
-	Exploits: Programas automáticos que aprovechan vulnerabilidades conocidas para comprometer un sistema
-	Payloads: Código a ejecutar una vez se accede a un sistema
-	Módulos Auxiliares: Herramientas para escaneo, fuzzing, etc.
-	Post-Explotación: Acciones a realizar para completar el proceso de pentesting

En este ejercicio, se utilizará su versión de consola, desde la cual es posible acceder a todas sus funcionalidades. Cabe destacar, además, que su uso fuera de entornos controlados sobre sistemas sin permiso es ilegal.

![msfconsole](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/4.msfconsole.jpg)

Como fin de la fase de escaneo, se procederá a identificar cada una de las posibles vulnerabilidades presentes en esta máquina.

La vulnerabilidad de versión 2.3.4 de vsftpd se muestra en el motor de búsqueda de Metasploit en la siguiente figura, la cual se define como Backdoor Command Execution.

![Exploit](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/5.exploit.jpg)

CVE-2011-2523, como se conoce esta vulnerabilidad, permite ejecutar comandos desde una shell remota en el puerto 6200/TCP cuando un usuario se autentica con un username en el que se contiene la secuencia “:)”. Que una vulnerabilidad sea backdoor supone la utilización de un método para acceder al sistema sin necesidad de pasar por los mecanismos de seguridad, lo que permite acceder al mismo cuando se quiera. Cabe destacar que dicho fallo no proviene del código fuente del paquete original, si no de una versión alterada que sustituyó al auténtico en la web oficial de vsftpd.

OpenSSH en su versión 4.7p1 también es conocida por su vulnerabilidad CVE-2008-5161.

Un atacante que intercepte el tráfico SSH cifrado puede, si envía ciertas peticiones especialmente formuladas, revelar los paquetes en texto plano, perdiendo así la confidencialidad de la sesión y abriendo la posibilidad de descifrar contraseñas y datos transferidos. Esta vulnerabilidad afecta a las versiones 4.7p1 y anteriores, debido al algoritmo de cifrado de datos que se usaba (por bloques en modo CBC).

No obstante, no existe ningún exploit en Metasploit para explotar dicha vulnerabilidad.

![Exploit Fail](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/6.exploit.jpg)

En último lugar, se estudiará la versión 3.0.20 de Samba.

![Exploit](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/7.exploit.jpg)

La versión 3.0.20 de Samba se ve afectada por la vulnerabilidad CVE-2007-2447, la cual permite la ejecución remota de comandos (Remote Code Execution). Dentro de Samba, existe un archivo de configuración llamado “username map”, el cual mapea usuarios de Windows a usuarios de Linux. No obstante, si en el campo de nombre de usuario del archivo mencionado se introducen comandos, se interpretan como un script, lo que permite la ejecución arbitraria o, como es el caso de este exploit, establecer una reverse shell.

# Explotación

Tras la fase de escaneo, se han determinado dos caminos a seguir desde Metasploit: atacar la vulnerabilidad de vsftpd o de Samba. En primer lugar, se ejecutará el exploit correspondiente al protocolo FTP.

Primero, se debe configurar el exploit.

![Exploit Config](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/8.exploit.jpg)

Con use 0 se selecciona el exploit deseado, en este caso, el único que aparece tras la búsqueda. Al ejecutar el comando show options nos muestra qué configuraciones son necesarias para que lanzar el exploit. Se aprecia que para este exploit hay que indicar únicamente el host objetivo (IP de la máquina), pues el puerto objetivo por defecto es correcto.

Una vez hecho, se procederá a ejecutar el exploit con el comando run.

![Exploit Run](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/9.exploit.jpg)

Al parecer, el exploit no funciona, al requerir unas credenciales que no se han obtenido. Más adelante, en el apartado 5.3.4 Informe y Lecciones Aprendidas se abordarán las causas del error.

Así pues, se continuará con el exploit del programa Samba en búsqueda de éxito. La configuración del exploit es similar anterior, debiendo indicar tanto el host objetivo como el local (máquina atacante).

![Exploit Samba](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/10.exploit.jpg)

Se procede a lanzar el exploit una vez configurado.

![Exploit Run](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/11.exploit.jpg)

Al parecer, no solo ha sido un éxito, sino que se ha logrado una shell como root, por lo que no hará falta escalar privilegios para obtener la root flag.

# Post Explotación

Con un poco de búsqueda, se obtienen ambas banderas.

![User Flag](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/12.userflag.jpg)

![Root Flag](https://alejandromartinezmoreno.github.io/HTB:Lame/Images/13.rootflag.jpg)

De este modo, se ha completado la máquina virtual Lame, por lo que procede estudiar las conclusiones de su realización.

# Informe y Lecciones Aprendidas

El interés en esta máquina virtual viene dado por el uso de la herramienta Metasploit. Metasploit es un framework de código abierto diseñado para desarrollar, probar y ejecutar exploits en sistemas vulnerables. Es una de las herramientas más conocidas en el mundo de la ciberseguridad y de las pruebas de penetración.

Gracias a su uso, se ha conseguido explotar una vulnerabilidad sin necesidad de entender realmente qué sucede o los detalles de la misma. Esa fácil accesibilidad convierte a Metasploit en un arma de doble filo, por lo que su uso fuera de entornos controlados de pruebas es perseguido.

Durante el desarrollo del desafío Capture The Flag, han aparecido las siguientes tácticas y técnicas calificadas por MITRE ATT&CK:
1. Reconocimiento (Reconnaissance)
1.1. T1595.002 – Active Scanning: Vulnerability Scanning. Se ha realizado un escaneo de puertos con Nmap para identificar puertos abiertos y versiones de sus servicios.

2. Acceso Inicial (Initial Access)
2.1. T1190 – Exploit Public-Facing Application. Se explota la vulnerabilidad de Samba mediante un exploit.

3. Ejecución (Execution)
3.1. T1059.004 – Command and Scripting Interpreter: Bash. Se han ejecutado comandos locales tras obtener acceso remoto al sistema.

4. Descubrimiento (Discovery)
4.1. T1082 – System Information Discovery. Se han usado comandos para identificar los usuarios activos y el entorno del sistema, como whoami, ls, cat.

5. Escalada de Privilegios (Privilenge Escalation)
5.1. T1068 – Exploitation for Priviledge Escalation. Se ha ejecutado un exploit que, gracias a una vulnerabilidad conocida, ha permitido el acceso como root.

Si bien Metasploit es una herramienta fuerte, para que su uso en la vida real sea eficaz se necesita identificar un protocolo o servicio cuya versión tenga una vulnerabilidad conocida. Por ello, se reitera la importancia de mantener los equipos, sobre todo aquellos que contengan información importante, con sus componentes y programas actualizados a sus últimas versiones.

[Volver al inicio](https://alejandromartinezmoreno.github.io/CyberHustler)

