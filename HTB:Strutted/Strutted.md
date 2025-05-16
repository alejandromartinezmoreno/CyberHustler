---
layout: default
---
# Máquina Virtual: Strutted
**Plataforma:** Hack The Box  
**Categoría:** Pentesting Web, Remote Code Execution, Escalada de Privilegios  

---

## Información General
- **IP de la máquina:** `10.10.11.59`
- **Objetivo:** Obtener dos flags: user y root
- **Herramientas utilizadas:** Nmap, BurpSuite, Netcat, etc.

---

## Resolución

# Reconocimiento

La IP objetivo será 10.10.11.59, mientras que la máquina atacante responderá a la IP 10.10.14.87.

# Escaneo

Trivialmente, se deberá realizar un escaneo de puertos con el objetivo de conocer, tanto qué puertos se encuentran abiertos, como qué protocolos se están llevando a cabo en cada puerto. Así, se hace uso de la herramienta Nmap:

![Escaneo Nmap](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/1.nmap.jpg)

El comando -sV indica la versión del protocolo que se está utilizando, mientras que con --min-rate-5000 se busca aminorar el tiempo de escaneo.

--min-rate (número) establece la tasa mínima de envío de paquetes por segundo que se envíaran durante el escaneo nmap. De este modo, con el comando anterior se ha asegurado una tasa de envío de paquetes por segundo de 5000, lo que supone un escaneo veloz. Por contraparte, supone que el escaneo sea ruidoso y fácil de detectar, aunque en entornos virtualizados y controlados no es algo a considerar.

La máquina objetivo contiene dos puertos TCP abiertos: el 22, en el que se encuentra activo el protocolo SSH, y el 80, con el protocolo HTTP.

Secure Shell (SSH) es un protocolo de red que permite el acceso a sistemas remotos de forma segura, utilizado para accesos a servidores y transferencias de archivos. Para acceder se necesita un usuario y una contraseña. Se está utilizando OpenSSH en su versión 8.9p1.

Por otro lado, las siglas HTTP se refieren a HyperText Transfer Protocol. Es el protocolo por excelencia de los servidores web: permite la transferencia de datos entre navegador y servidor. El servidor web utilizado es Nginx en su versión 1.18.0.

¿Existe alguna vulnerabilidad en las versiones de las herramientas usadas? Para OpenSSH 8.9 no se conocen vulnerabilidades críticas hasta la fecha, mientras que para Nginx 1.18.0 las vulnerabilidades conocidas requieren de ciertos pasos previos para su explotación.

Por tanto, el siguiente paso a dar será examinar el servidor web, con el objetivo de encontrar credenciales que permitan el acceso al protocolo SSH, donde, comúnmente, se esconde las flags. Para ello, añadiremos la IP a la lista de hosts definidos y buscaremos la página web en cualquier navegador. No es obligatorio añadir hosts a la lista para completar el reto, aunque es recomendable para trabajar cómodamente.

![Host Añadido](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/2.host.jpg)

![HTTP](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/3.http.jpg)

Parece ser una página web dedicada a crear links con los que compartir imágenes previamente subidas al servidor.

En la esquina superior derecha se puede apreciar el apartado Dowload. Al hacer click, se descargará una carpeta con diferentes archivos de configuración del contenedor en uso, gestionado por Docker.

Docker es una plataforma dedicada a creación, distribución y ejecución de aplicaciones en contenedores.

Entre varios archivos, el primero que salta a la vista es el llamado Dockerfile, el cual es un script que Docker utiliza para construir la imagen del contenedor.

![Dockerfile](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/4.dockerfile.jpg)

El documento nos aporta cierta información:
- La imagen base está basada en Alpine Linux y utiliza Maven para compilar la aplicación Java.
- El directorio de trabajo se establece en /tmp/strutted.
- Utiliza un servidor Tomcat en su versión 9.0.

Dentro de la carpeta, también destaca el archivo tomcat-users-xml.

![tomcat-users.xml](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/5.tomcatusers.jpg)

En este archivo se define un usuario admin con permisos especiales, al igual que aparece su contraseña en texto plano. Sin embargo, dichas credenciales no sirven para acceder al protocolo SSH.

![SSH denegado](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/6.sshdenial.jpg)

Como último archivo destacable, se encuentra el archivo pom.xml. Un archivo POM (Project Object Model) define la configuración principal de Maven, por lo que podría haber algo de interés.

![pom.xml](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/7.pom.jpg)

Dentro del archivo, destaca el apartado properties, el cual especifica las variables reutilizables dentro del proyecto Maven. Tras hacer un estudio rápido de cada línea y variable, se aprecia una vulnerabilidad. El framework utilizado es Struts2 en su versión 6.3.0.1.

Struts2 es un framework orientado al desarrollo de aplicaciones web en Java. Existe una vulnerabilidad conocida para la versión 6.3.0.1, conocida como CVE-2024-53677. La lógica de carga de los archivos en este framework contiene una vulnerabilidad que permite la subida sin restricciones de cualquier fichero. Así, explotádola, se abrirá la posibilidad de subir archivos maliciosos o scripts con los que comprometer el sistema, por lo que comienza la fase de explotación.

# Explotación

La herramienta Burp Suite, definida en el apartado 4.2 del presente trabajo, es de gran utilidad en este momento: permite interceptar peticiones y modificarlas las veces que se requiera. El primer paso será interceptar la petición POST que tiene lugar al subir un archivo.

![Burp Suite](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/8.burp.jpg)

![Burp Suite](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/9.burp.jpg)

La primera figura muestra la petición que el cliente hace a la hora de subir un archivo, en este caso, una imagen .jpeg a modo de ejemplo. El contenido de la imagen se encuentra borrado, a modo de hacer más simple la comprensión, ya que dicho contenido no afectará a la respuesta del servidor.

Por otro lado, la respuesta del servidor contenida en la segunda figura muestra en pantalla, entre otras cosas, un mensaje de éxito en la subida de archivos y el parámetro img_src, que indica el directorio del servidor en el que se guardará dicho archivo.

El servidor solamente acepta archivos .jpg, .jpeg, .png y .gif, aunque, gracias a la vulnerabilidad previamente descrita, existe una manera de engañar al servidor e introducir cualquier extensión.

El método POST de HTTP permite enviar más de un archivo. Por tanto, el exploit consistirá en “colar” un archivo .jsp (Java Server Pages) en el que irá contenido un script encargado de abrir una terminal de comandos en el servidor.

Para poder explotar la vulnerabilidad correctamente, habrá que:
- Cambiar la primera letra de upload a mayúscula.
- Añadir un segundo archivo de la forma mostrada en la Figura 10, añadiendo en el campo name “top.UploadFileName” y omitiendo el campo filename.

![Burp Suite](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/10.burp.jpg)

En la Figura 11 es apreciable que la explotación ha sido un éxito: se ha introducido en el servidor un archivo JavaServer Pages. Sin embargo, dicho archivo no es accesible y, por tanto, se ha de modificar de nuevo la petición. En esta tercera petición:
- Se añadirá el código para crear una PowerShell directamente a la petición.
- Se realizará un Path Traversal, de modo que la ruta al archivo sea accesible. 

![Burp Suite](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/11.burp.jpg)

![Burp Suite](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/12.burp.jpg)

De este modo, el archivo se ha subido correctamente en un directorio distinto y accesible: http://strutted.htb/shell.jsp

![whoami](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/13.whoami.jpg)

![ls](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/14.ls.jpg)

En las capturas se aprecian las ejecuciones de los comandos whoami y ls. Whoami indica bajo qué usuario se ejecuta la terminal, en este caso, el usuario tomcat. Por otro lado, ls permite ver los archivos y carpetas que contiene el directorio. Cabe además destacar, a modo de curiosidad, que la información de la imagen que no ha sido borrada aparece como texto plano en la interfaz de la shell remota.

Sin embargo, esta terminal tiene accesos clave bloqueados, como, por ejemplo, el protocolo SSH. Por ello, el siguiente avance será establecer una reverse shell: establecer una conexión entre máquina atacante y víctima que permita ejecutar comandos como si tomaran lugar en el sistema objetivo. Es bastante útil a la hora de bypassear filtros del Firewall y otros accesos bloqueados.

1. Crear un servidor HTTP en la máquina atacante en el puerto 80:

![Servidor HTTP Puerto 80](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/15.httpserver.jpg)

2. Crear archivo en el que se contenga un script que abra una conexión inversa (reverse shell) entre ambas máquinas:

![Script Reverse Shell](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/16.script.jpg)

Se usará el puerto 443, al no estar comúnmente en uso y, por tanto, no soler estar bloqueado por el firewall.

3. Descargar el archivo shell.sh en la máquina víctima, gracias al servidor previamente creado, con el comando wget:

![Wget](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/17.wget.jpg)

![Shell.sh](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/18.shell.jpg)

4. Ejecutar NetCat para escuchar conexiones entrantes por el puerto 443, y así establecer la conexión inversa:

![Netcat](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/19.netcat.jpg)

En caso de seguir los pasos de manera correcta, se creará la reverse shell. La terminal está abierta bajo el usuario tomcat, el cual no goza de privilegios superiores. Por tanto, se deberán descubrir otros usuarios cuyas credenciales permitan el acceso al protocolo SSH. En un sistema Linux, la información sobre los usuarios esta contenida en el archivo /etc/passwd, por lo que se podrá ver su contenido si se utiliza el comando cat /etc/passwd.

![/etc/passwd](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/20.etcpasswd.jpg)

Salta a la vista el usuario james, el cual es definido como administrador de la red, por lo que seguramente tenga acceso a ciertos privilegios o, mejor aún, sea el usuario desde el que se lleva a cabo el protocolo SSH.

Dentro de la carpeta config existe un archivo llamado tomcat-users.xml. Pese a que comparte nombre con el archivo que se vió en el inicio su contenido es diferente.

![tomcat-users.xml](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/21.tomcatusers.jpg)

Dentro del archivo se encuentra contenida una contraseña para el usuario admin. Al ser el usuario james administrador, probar ambas credenciales para acceder al protocolo SSH parece ser un buen paso que dar.

![User Flag](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/22.userflag.jpg)

Las credenciales usadas son válidas y permiten el acceso al protocolo SSH bajo el usuario james. Ahí se encuentra, además la user flag.

# Post Explotación

En este punto, el sistema ya se encuentra comprometido y se ha logrado el acceso a él mediante el usuario james. El siguiente objetivo es obtener la root flag, la cual, para acceder a ella, será necesario llevar a cabo una escalada de privilegios.

![Privilegios de james](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/23.james.jpg)

El comando sudo -l permite conocer qué privilegios tiene el usuario activo en la terminal. En este caso, james puede utilizar tcpdump como root sin necesidad de contraseña.

Existe un repositorio en Github llamado GTFOBins que presenta un listado de binarios preinstalados en Linux útiles para bypassear sistemas de seguridad mal configurados.

Para tcpdump, encontramos una serie de comandos que permiten escalar priviliegios, siempre el binario esté permitido para ejecutarse como superusuario. Es decir, como se cumplen las condiciones, se podrán ejecutar comandos arbitrarios como usuario root si se explota esta vulnerabilidad.

![GTFOBins](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/24.gtfobins.jpg)

Lo que sucederá una vez se ejecute esté comando es la creación de un archivo ejecutable que contenga el comando id, el cual muestra el usuario actual, y se llamará a dicho script como root mediante tcpdump con sudo.

Sin embargo, este comando no abrirá una nueva terminal como el usuario root, por lo que hay que elegir un comando que guardar que permita lograr dicho fin.

![Escalada de privilegios](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/25.ep.jpg)

El comando cp /bin/bash /tmp/htb_root copia el contenido de /bin/bash, el cual permite abrir una terminal en Linux, al archivo previamente creado htb_root, el cual se encuentra en /tmp/, carpeta accesible y modificable por cualquier usuario.

Por otro lado, con el comando chmod +s /tmp/htb_root, se busca activa el bit setuid, lo que permite que un archivo se ejecute como usuario root.

Una vez ejecutada esta serie de comandos, se procederá a comprobar que, efectivamente, ahora el archivo htb_root tiene el bit setuid activado.

![Bash como root](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/26.root.jpg)

La “s” en rwsr-sr-x indica que, efectivamente, dicho bit esta activado, por lo que se procederá a ejecutar el archivo. La razón de añadir -p al comando es asegurar que dicho comando se ejecutará bajo el usuario root, pues, por defecto, Linux tenderá a ofrecer los mínimos privilegios posibles.

Exitosamente se ha logrado rootear el sistema, por lo que, simplemente queda encontrar la root flag. Al parecer, el directorio activo en la terminal no es /root, donde suele encontrarse la bandera restante, por lo que, con esperanza de encontrar dicha flag, se accederá a dicho directorio.

![Root flag](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado/HTB:Strutted/Images/27.rootflag.jpg)

De este modo termina el desafío Capture The Flag: Strutted, y se procederá a comentar las conclusiones extraídas durante el desarrollo de la máquina virtual.

# Informe y Lecciones Aprendidas

El desafío Strutted viene caracterizado por una vulnerabilidad de Apache Struts2, un framework de código abierto, en su versión 6.3.0.1. Dicha vulnerabilidad se conoce como CVE-2024-53677 y, si se explota, se puede conseguir la subida sin restricciones de ficheros, incluyendo aquellos maliciosos. Desde la versión 2.0.0 hasta la 6.4.0 existe un fallo en el componente encargado de manejar las cargas de archivos (FileUploadInterceptor), que se ha logrado explotar en la resolución del CTF.

Las tácticas y técnicas utilizadas en el proceso de resolución del desafío se definen según el estándar MITRE ATT&CK:
1. Reconocimiento (Reconnaissance)
1.1. T1595.002 – Active Scanning: Vulnerability Scanning. Se ha realizado un escaneo de puertos con Nmap para identificar puertos abiertos y versiones de sus servicios.

2. Acceso Inicial (Initial Access)
2.1. T1190 – Exploit Public-Facing Application. Se explota la vulnerabilidad de Apache Struts2 para subir un archivo JSP malicioso.

3. Ejecución (Execution)
3.1. T1059.004 – Command and Scripting Interpreter: Bash. Se ha ejecutado una reverse shell desde el servidor mediante un JSP malicioso.

4. Comando y Control (Command and Control)
4.1. T1105 – Ingress Tool Transfer. Se han transferido archivos desde la máquina atacante, como shell.sh.
4.2. T1071.001 – Application Layer Protocol: Web Protocol. Se ha usado HTTP para interactuar con la reverse shell.

5. Acceso a Credenciales (Credential Access)
5.1. T1552.001 – Unsecured Credentials in Files. Se han extraído credenciales en texto plano desde el archivo tomcat-users.xml.

6. Escalada de Privilegios (Privilenge Escalation)
6.1. T1548.003 – Abuse Elevation Control Mechanism: Sudo and Sudo Caching. Se han usado técnicas de GTFOBins para obtener una shell como root mediante comandos sudo.

7. Descubrimiento (Discovery)
7.1. T1082 – System Information Discovery. Se han usado comandos para identificar los usuarios activos y el entorno del sistema, como whoami, ls, cat.

Para estudiar el impacto que una vulnerabilidad de este tipo puede tener, véase el caso de Equifax, una de las tres principales agencias de calificación crediticia de Estados Unidos. En 2017, Equifax sufrió una brecha que permitió a los atacantes robar datos personales de 147 millones de personas. Usaban Apache Struts2, y no parchearon la versión del framework, por tanto, los actores tuvieron explotaron la misma vulnerabilidad que se ha visto durante el desarrollo del desafío.

En la vida cotidiana, mantener el sistema operativo de un dispositivo desactualizado puede suponer ser víctima de una brecha de seguridad. Por tanto, mensaje para el lector: actualizar los programas a las últimas versiones lanzadas tiene más importancia de la que aparenta si quieres asegurar la confidencialidad, integridad y disponibilidad de tus datos.

[Volver al inicio](https://alejandromartinezmoreno.github.io/Trabajo-Fin-de-Grado)
