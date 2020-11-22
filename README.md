# Guia para la instalación y configuración de LAMP en Amazon Linux 2

En esta guia vamos a seguir los pasos necesarios para instalar un servidor LAMP en una máquina Amazon Linux 2 alojada en AWS mediante una conexión SSH, y después haremos las configuraciones necesarias para crear varias páginas virtuales y securizar alguna parte de la página.

# Instalación

## Pasos previos

Antes de ponernos a instalar nuestro servidor LAMP o cualquier otra cosa, lo primero que tenemos que hacer siempre en una máquina Linux al iniciarla por primera vez es actualizar los repositorios. Para ello ejecutaremos este comando en la consola.

~~~
$ sudo yum update -y
~~~

## Instalacion y arranque del servidor

Ahora necesitaremos instalar una serie de paquetes para que nuestro servidor pueda funcionar, para ello las instalaremos con los siguientes comandos:

### Añadir los repositorios
~~~
$ sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
~~~

### Instalar los paquetes
~~~
$ sudo yum install -y httpd mariadb-server
~~~

### Arrancar el servidor, activar autoarranque y comprobar
~~~
$ sudo systemctl start httpd

$ sudo systemctl enable httpd
> Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.

$ sudo systemctl is-enabled httpd
> enabled
~~~

### Configurar base de datos MariaDB

Ahora vamos a hacer una instalacion correcta de la base de datos MariaDB, para ello vamos a levantar el servidor y vamos a hacer lo que se llama una instalación segura definiendo una contraseña para el administrador. Para ello vamos a primero iniciar el servidor con el siguiente comando:

~~~
$ sudo systemctl start mariadb
~~~

Y a continuación vamos a ejecutar este otro comando, el cual nos va a pedir un prompt, le daremos Y de sí, y nos hara otro de la contraseña actual, la cual vamos a dejar vacía puesto que no hay y seguidamente vamos a escribir 2 veces la nueva contraseña, una vez hecho esto le daremos a enter hasta que acabe de preguntarnos cosas (a todo que sí).

~~~
$ sudo mysql_secure_installation
~~~

Ahora de forma opcional podemos hacer que MariaDB se inicie cada vez que iniciamos el ordenador, yo lo prefiero así por lo que voy a dejarlo activado con el siguiente comando:

~~~
$ sudo systemctl enable mariadb
~~~

### Instalar phpMyAdmin

phpMyAdmin es un gestor de base de datos online que nos va a venir muy bien para gestionar nuestra base de datos MariaDB, por lo que lo vamos a instalar.

Para instalarlo vamos a empezar con las dependencias que tendremos que instalar para un correcto funcionamiento de phpMyAdmin, las cuales instalaremos con el siguiente comando:

~~~
$ sudo yum install php-mbstring -y
~~~

Una vez instalado vamos a reiniciar los servicios de **httpd** y **php-fpm**

~~~
$ sudo systemctl restart httpd
$ sudo systemctl restart php-fpm
~~~

Ahora vamos a navegar dentro de nuestra carpeta raiz de la web **/var/www/html** y vamos a descargar la ultima version de phpMyAdmin

~~~
$ cd /var/www/html
$ wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.tar.gz
~~~

Ahora que la hemos descargado vamos a extraerla en una carpeta llamada phpMyAdmin y vamos a eliminar el archivo comprimido que descargamos ya que ya lo hemos descomprimido.

~~~
$ mkdir phpMyAdmin && tar -xvzf phpMyAdmin-latest-all-languages.tar.gz -C phpMyAdmin --strip-components 1
$ rm phpMyAdmin-latest-all-languages.tar.gz
~~~

Una vez hecho esto ya deberiamos poder tener acceso a phpMyAdmin, pero para ello tendremos primero que reiniciar el servidor apache e iniciar el de MariaDB si no lo teniamos previamente.

~~~
$ sudo systemctl restart httpd
$ sudo systemctl start mariadb
~~~

Una vez hecho todo lo anterior en teoría tendremos terminado el proceso si lo hemos hecho todo bien por lo que al acceder a nuestro servidor desde el navegador con la ruta **/phpMyAdmin** ya podremos acceder a el phpMyAdmin que acabamos de instalar y nos aparecería una pantalla como la que podemos ver en la zona inferior, en la cual iniciaremos sesión con el usuario **root** y la contraseña que hayamos puesto anteriormente.

![Resultado página phpMyAdmin](https://i.imgur.com/ktzwoJJ.png)


Una vez hecho esto ya tendremos todos los paquetes necesarios de un servidor LAMP instalado, y ya podemos proceder a las comprobaciones y configuraciones.

# Configuracion de los sitios Web

## Comprobacion de la instalación

Ahora que ya hemos instalado y arrancado nuestro servidor LAMP ya deberiamos estar ofreciendo un servicio web en este mismo momento, Apache a modo de test desde el momento en el que lo lanzamos por primera vez empieza a ofrecer una web de prueba sin necesidad de configuraciones para que podamos comprobar de forma rápida al visitarla si nuestra instalacion ha sido correcta o hemos cometido algún fallo.

Para comprobar esta web tendremos que acceder desde un navegador a la dirección de nuestra máquina AWS mediante HTTP con un enlace como este: _http://ec2-52-206-214-19.compute-1.amazonaws.com/_

![Captura web Apache prueba](https://i.imgur.com/veu52RH.png)

Si nos aparece una web como la que podemos ver en la imagen superior significa que lo hemos hecho todo bien.

## Configuración de una web virtual

Lo primero que vamos a hacer antes de ponernos a configurar nuestras webs, vamos a cambiar los permisos de los directorios donde vamos a almacenar nuestras webs, en este caso el directorio raiz de las webs es **/var/www/**

### Añadirnos al grupo apache

Nosotros vamos a querer cambiar los permisos de los directorios y subdirectorios donde vamos a almacenar todos los documentos a nuestro usuario y el grupo apache. Por lo que lo primero que tendremos que hacer va a ser añadirnos al grupo apache, para ello ejecutaremos los siguientes comandos:

~~~
$ sudo usermod -a -G apache ec2-user
~~~

y para comprobar si hemos sido añadidos al grupo correctamente cerraremos sesión con el comando `exit` y volveremos a conectarnos a la máquina, una vez conectados ejecutaremos el siguiente comando para comprobar de que grupos formamos parte:

~~~
$ groups
> ec2-user adm wheel apache systemd-journal
~~~


Si aparece el grupo apache en el listado como en el ejemplo de la zona superior significará que nos hemos añadido correctamente al grupo, por lo que ya podemos cambiar los permisos del directorio raiz.

### Cambiar permisos en el directorio raiz.

Para cambiar los permisos del directorio raiz vamos a ejecutar los siguientes comandos:

#### Cambiar la propiedad a el usuario ec2-user y el grupo apache
~~~
$ sudo chown -R ec2-user:apache /var/www
~~~

#### Cambiar los permisos de todos los directorios
~~~
$ sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
~~~

#### Cambiar los permisos de todos los ficheros
~~~
$ find /var/www -type f -exec sudo chmod 0664 {} \;
~~~

### Crear una web virtual

Ahora que ya hemos configurado nuestro espacio de trabajo, vamos a crear una página web virtual, para ello nos colocaremos dentro del directorio web raiz **/var/www/** y crearemos una carpeta donde almacenaremos el contenido de nuestra web.

~~~
$ cd /var/www/
$ mkdir diputacion
~~~

Ahora crearemos un pequeño archivo html para probar que funciona

~~~
$ echo "<h1>Diputación de Castellón</h1>" > diputacion/index.html
~~~

### Configurar web virtual

Ahora una vez creada nuestra web de prueba vamos a configurar este sitio web para que podamos acceder a el, para ello vamos a crear un archivo de configuracion con el mismo nombre que nuestra web

~~~
$ sudo nano /etc/httpd/conf.d/diputacion.conf
~~~

y escribiremos en el el siguiente contenido:

~~~
<VirtualHost *:<puerto>>
  DocumentRoot <ubicación>
</VirtualHost>
~~~

**NOTA:** _Si el puerto que vamos a utilizar es diferente del 80 hay que añadir un listener escribiendo esta línea en la primera línea de el archivo de configuracion_ 

~~~
Listen <puerto>
~~~

Ahora comprobaremos la sintaxis con el siguiente comando y si no tenemos ningun error reiniciaremos el servidor para ver los cambios.

~~~
$ sudo apachectl configtest
> Syntax OK

$ sudo systemctl restart httpd
~~~

Si lo hemos hecho todo correctamente hasta ahora deberiamos ver la pagina web de esta forma

![Página web virtual de prueba](https://i.imgur.com/DlQbJrT.png)

### Configurar seguridad tipo Basic

Ahora que ya hemos comprobado nuestra primera página virtual, ahora vamos a configurar una securización de tipo basic, para ello vamos a crear una directorio nuevo dentro de nuestra carpeta diputación llamada administracion

~~~
$ cd /var/www/diputacion/
$ mkdir administracion
$ echo "<h1>Página de administración</h1>" > administracion/index.html
~~~

Ahora vamos a crear un archivo de autenticaciones basic, el cual almacenaremos dentro de la carpeta de configuraciones httpd, dentro de un subdirectorio al que llamaremos passwords, por lo que a continuacion vamos a crear el directorio **passwords** y a crear el archivo de autenticación de tipo **basic**

~~~
$ sudo mkdir /etc/httpd/passwords
$ sudo htpasswd -c /etc/httpd/passwords/passwords-basic admin 
~~~

Al ejecutar el comando `htpasswd` nos pedira que escribamos una contraseña 2 veces, escribiremos la misma ya que es para comprobar que la hayamos escrito correctamente y una vez hecho eso ya tendremos creado el archivo de contraseñas en el subdirectorio que hemos creado con el usuario admin y la contraseña que le hayamos puesto.

Ahora tendremos que configurar esta autenticacion en el fichero de configuración que hemos creado previamente en /etc/httpd/conf.d/diputacion.conf, para ello abriremos el fichero con nano con el siguiente comando:

~~~
$ sudo nano /etc/httpd/conf.d/diputacion.conf
~~~

y le añadiremos las siguientes líneas: 

~~~
<Directory "/var/www/diputacion/administracion">
  AuthType Basic
  AuthName "Zona Administración"
  AuthUserFile "/etc/httpd/passwords/passwords-basic"
  Require valid-user
</Directory>
~~~

Saber que en estas líneas he escrito la ubicación del directorio a securizar de mi caso especifico y que tanto la ubicación como el AuthName que aparecen en el ejemplo han de sustituirse con los que se requieran con su uso particular.

Finalmente para comprobar que no hayan errores de sintaxis y para aplicar los cambios escribiremos los siguientes comandos:

~~~
$ sudo apachectl configtest
> Syntax OK

$ sudo systemctl restart httpd
~~~

Si lo hemos hecho todo de forma correcta, ahora al acceder a nuestra página desde el navegador con el sufijo **/administracion** debería salirnos este dialogo de autenticación, el cual requiere que nos autentiquemos para poder acceder

![Página nos pide autenticación](https://i.imgur.com/5TQEQEP.png)

Y ahora comprobaremos que funcione la autenticación escribiendo las credenciales de autenticacion que hemos creado anteriormente en nuestro caso son **admin:admin** y si lo hemos hecho todo bien deberiamos tener como resultado la página que se ve en la imagen inferior.

![Resultado página administración](https://i.imgur.com/o5en52J.png)

### Configuración tipo Digest

Una vez hemos probado la seguridad de tipo basic, ahora vamos a crear otra subpágina dentro de nuestra página de la diputación, la cual vamos a securizar con Digest, otro tipo de seguridad mejor que la de basic. Para ello crearemos una nueva carpeta dentro de la raiz del documento llamada enchufes y crearemos en ella un *index.html*

~~~
$ cd /var/www/diputacion/
$ mkdir enchufes

$ echo "<h1>Enchufada del mes: Alexia Agramunt</h1>" > enchufes/index.html
~~~

Ahora tendremos que crear el archivo de claves tal y como hicimos en el basic, a diferencia de que en este caso tendremos que utilizar el comando `htdigest` y le añadiremos un apartado más el cual será el nombre de la autenticación.

**NOTA:** _Es importante recordar el nombre que le pongamos ya que después lo necesitaremos al securizar el directorio._

~~~
$ sudo htdigest -c /etc/httpd/passwords/passwords-digest "ENCHUFES" fernando
> Adding password for fernando in realm ENCHUFES.
~~~

Ahora que ya tenemos nuestro usuario fernando creado vamos a editar de nuevo el archivo de configuracion de nuestra web.

~~~
$ sudo nano /etc/httpd/conf.d/diputacion.conf
~~~

Y le añadiremos el siguiente contenido:

~~~
<Directory "/var/www/diputacion/enchufes">
  AuthType Digest
  AuthName "ENCHUFES"
  AuthUserFile "/etc/httpd/passwords/passwords-digest"
  Require valid-user
</Directory>
~~~

Y finalmente comprobaremos la sintaxis y reiniciaremos el servidor con los siguientes comandos:

~~~
$ sudo apachectl configtest
> Syntax OK

$ sudo systemctl restart httpd
~~~

Si lo hemos hecho todo bien hasta ahora deberiamos ser agradecidos con el dialogo de autenticación al intentar navegar a la página **/enchufes**

![Diálogo de autenticación digest](https://i.imgur.com/lLw3kHy.png)

Y si aplicamos nuestras credenciales, **fernando:fernando** en nuestro caso, y lo hemos hecho todo bien deberiamos poder acceder a la página y observar el resultado.

![Resultado autenticación digest](https://i.imgur.com/QoyeRuS.png)

Y hasta aquí con esta guía, hemos aprendido todo lo necesario para aprobar el examen.