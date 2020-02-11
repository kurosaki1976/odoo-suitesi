# Instalación y configuración de la Suite de Seguridad Informática en Debian 9 Stretch

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## ¿Qué es la Suite Seguridad Informática?

La Suite de Seguridad Informática (Suite SI) es un proyecto para la gestión de la seguridad a nivel empresarial, desarrollado por la división de DeSoft en Las Tunas. Esta aplicación permite la automatización de los servicios, la vigilancia y control sobre el software y hardware que conforman el patrimonio informático de una institución. A juicio de sus creadores, tiene como valor agregado promover una mayor cultura de los empresarios y funcionarios hacia la importancia de la seguridad informática. La aplicación está soportada en la versión 8 de Odoo y necesita OCSInventory para el sondeo de hardware/software de los hosts que integran la red.

## Breve introducción a OCSInventory NG

`Open Computer and Software Inventory Next Generation` es un software libre que permite a los Administradores de Tecnologías de la Információn gestionar el inventario de sus activos de TI. OCS-NG recopila información sobre el hardware y software de equipos que hay en la red que ejecutan el programa de cliente OCS `OCS Inventory Agent`. Puede utilizarse para visualizar el inventario a través de una interfaz web. Además, comprende la posibilidad de implementación de aplicaciones en los equipos de acuerdo a criterios de búsqueda, escanear la red por medio del IPDiscovery, o instalar aplicaciones remotamente creando paquetes de instalación.

OCSInventory-NG funciona con el servidor web `Apache2` y el gestor de base de datos `MySQL`.

## Instalación de paquetes necesarios

```bash
apt install apache2 libapache2-mod-perl2 libxml-simple-perl libcompress-zlib-perl libdbi-perl libdbd-mysql-perl libapache-dbi-perl libnet-ip-perl libsoap-lite-perl libarchive-zip-perl make build-essential php-pclzip php7.0 libapache2-mod-php7.0 php7.0-common php-pear php7.0-cli php7.0-ldap libxml-simple-perl libio-compress-lzma-perl php7.0-gd php7.0-curl php-soap libsoap-lite-perl libmojolicious-perl libplack-perl php-mbstring php-zip -y
```

Instalar dependencias CPAN

```bash
cpan
install CPAN
reload cpan
install XML::Entities
exit
```

Crear certificado TLS autofirmado y asignar permisos necesarios

```bash
openssl req -x509 -sha512 -days 3650 -nodes \
	-subj "/C=CU/ST=Provincia/L=Ciudad/O=Organización/OU=IT/CN=SuiteSI/emailAddress=postmaster@dominio.cu/" \
	-reqexts SAN -extensions SAN -config <(cat /etc/ssl/openssl.cnf \
		<(printf "\n[SAN]\nsubjectAltName=DNS:suitesi.dominio.cu,DNS:inventory.dominio.cu,\
		DNS:localhost,IP:127.0.0.1")) \
	-newkey rsa:2048 \
	-keyout /etc/ssl/private/SuiteSI.key \
	-out /etc/ssl/certs/SuiteSI.crt

chmod 0444 /etc/ssl/certs/SuiteSI.crt
chmod 0400 /etc/ssl/private/SuiteSI.key
```

Comprobar correcta creación del certificado

```bash
openssl x509 -in /etc/ssl/certs/SuiteSI.crt -text -noout
```

Establecer zona horaria para `PHP`

```bash
cp /etc/php/7*/apache2/php.ini{,.org}
sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/" /etc/php/7*/apache2/php.ini
```

### Servidor de bases de datos

Instalar servidor de base de datos MySQL

```bash
apt install mysql-server php-mysql -y
```

Configurar base de datos para OCSInventory-NG

```bash
mysql -u root
	SET PASSWORD FOR 'root'@'localhost' = PASSWORD('MYSQL_ROOT_PASSWORD');
	CREATE DATABASE ocsinventory_db;
	CREATE USER 'ocsuser'@'%' IDENTIFIED BY 'OCSUSER_PASSWORD';
	GRANT ALL PRIVILEGES ON ocsinventory_db.* TO 'ocsuser'@'localhost' WITH GRANT OPTION;
	FLUSH PRIVILEGES;
quit
```

### Instalación y configuración de OCSInventory-NG

Descargar el paquete e instalarlo

```bash
wget https://github.com/OCSInventory-NG/OCSInventory-ocsreports/releases/download/2.3.1/OCSNG_UNIX_SERVER-2.3.1.tar.gz
tar -xzmf OCSNG_UNIX_SERVER-2.3.1.tar.gz -C /usr/src/
cd /usr/src/OCSNG_UNIX_SERVER-2.3.1
sh setup.sh
```

* Nota: Se puede ir presionando la tecla `Enter` hasta finalizar, pero es recomendable ir leyendo cada una de las opciones de configuración que se presentan.

Editar los ficheros de configuración de OCSInventory-NG

```bash
mv /etc/apache2/conf-available/ocsinventory-reports.conf{,.org}
mv /etc/apache2/conf-available/z-ocsinventory-server.conf{,.org}
```

```bash
nano /etc/apache2/conf-available/ocsinventory-reports.conf

Alias /ocsreports /usr/share/ocsinventory-reports/ocsreports
<Directory /usr/share/ocsinventory-reports/ocsreports>
	<IfModule mod_authz_core.c>
		Require all granted
	</IfModule>
	<IfModule !mod_authz_core.c>
		Order deny,allow
		Allow from all
	</IfModule>
	Options Indexes FollowSymLinks
	DirectoryIndex index.php
	AllowOverride Options
    <IfModule mod_php5.c>
        AddType application/x-httpd-php .php
        php_flag file_uploads           on
        php_value post_max_size         101m
        php_value upload_max_filesize   100m
        php_flag magic_quotes_gpc      off
    </IfModule>
    <IfModule mod_php7.c>
        AddType application/x-httpd-php .php
        php_flag file_uploads           on
        php_value post_max_size         101m
        php_value upload_max_filesize   100m
        php_flag magic_quotes_gpc      off
    </IfModule>
</Directory>

Alias /download /var/lib/ocsinventory-reports/download
<Directory /var/lib/ocsinventory-reports/download>
	<IfModule mod_authz_core.c>
		Require all granted
	</IfModule>
	<IfModule !mod_authz_core.c>
		Order deny,allow
		Allow from all
	</IfModule>
</Directory>

Alias /snmp /var/lib/ocsinventory-reports/snmp
<Directory /var/lib/ocsinventory-reports/snmp>
	<IfModule mod_authz_core.c>
		Require all granted
	</IfModule>
	<IfModule !mod_authz_core.c>
		Order deny,allow
		Allow from all
	</IfModule>
</Directory>
```

```bash
nano /etc/apache2/conf-available/z-ocsinventory-server.conf

<IfModule mod_perl.c>
	PerlSetEnv OCS_MODPERL_VERSION 2
	PerlSetEnv OCS_DB_HOST 127.0.0.1
	PerlSetEnv OCS_DB_PORT 3306
	PerlSetEnv OCS_DB_NAME ocsinventory_db
	PerlSetEnv OCS_DB_LOCAL ocsinventory_db
	PerlSetEnv OCS_DB_USER ocsuser
	PerlSetVar OCS_DB_PWD OCSUSER_PASSWORD
	PerlSetEnv OCS_OPT_LOGPATH "/var/log/ocsinventory-server"
	PerlSetEnv OCS_OPT_DBI_PRINT_ERROR 0
	PerlSetEnv OCS_OPT_UNICODE_SUPPORT 1
	PerlAddVar OCS_OPT_TRUSTED_IP 127.0.0.1
	PerlSetEnv OCS_OPT_WEB_SERVICE_ENABLED 0
	PerlSetEnv OCS_OPT_WEB_SERVICE_RESULTS_LIMIT 100
	PerlSetEnv OCS_OPT_OPTIONS_NOT_OVERLOADED 0
	PerlSetEnv OCS_OPT_COMPRESS_TRY_OTHERS 1
	PerlSetEnv OCS_OPT_LOGLEVEL 0
	PerlSetEnv OCS_OPT_PROLOG_FREQ 12
	PerlSetEnv OCS_OPT_INVENTORY_ON_STARTUP 0
	PerlSetEnv OCS_OPT_AUTO_DUPLICATE_LVL 15
	PerlSetEnv OCS_OPT_SECURITY_LEVEL 0
	PerlSetEnv OCS_OPT_LOCK_REUSE_TIME 600
	PerlSetEnv OCS_OPT_TRACE_DELETED 0
	PerlSetEnv OCS_OPT_FREQUENCY 0  
	PerlSetEnv OCS_OPT_INVENTORY_DIFF 1
	PerlSetEnv OCS_OPT_INVENTORY_TRANSACTION 1
	PerlSetEnv OCS_OPT_INVENTORY_WRITE_DIFF 1
	PerlSetEnv OCS_OPT_INVENTORY_CACHE_ENABLED 1
	PerlSetEnv OCS_OPT_INVENTORY_CACHE_REVALIDATE 7
	PerlSetEnv OCS_OPT_INVENTORY_CACHE_KEEP 1
	PerlSetEnv OCS_OPT_DOWNLOAD 0
	PerlSetEnv OCS_OPT_DOWNLOAD_PERIOD_LENGTH 10
	PerlSetEnv OCS_OPT_DOWNLOAD_CYCLE_LATENCY 60
	PerlSetEnv OCS_OPT_DOWNLOAD_FRAG_LATENCY 60
	PerlSetEnv OCS_OPT_DOWNLOAD_GROUPS_TRACE_EVENTS 1
	PerlSetEnv OCS_OPT_DOWNLOAD_PERIOD_LATENCY 60
	PerlSetEnv OCS_OPT_DOWNLOAD_TIMEOUT 7
	PerlSetEnv OCS_OPT_DOWNLOAD_EXECUTION_TIMEOUT 120
	PerlSetEnv OCS_OPT_DEPLOY 0
	PerlSetEnv OCS_OPT_ENABLE_GROUPS 1
	PerlSetEnv OCS_OPT_GROUPS_CACHE_OFFSET 43200
	PerlSetEnv OCS_OPT_GROUPS_CACHE_REVALIDATE 43200
	PerlSetEnv OCS_OPT_IPDISCOVER 2
	PerlSetEnv OCS_OPT_IPDISCOVER_BETTER_THRESHOLD 1
	PerlSetEnv OCS_OPT_IPDISCOVER_LATENCY 100
	PerlSetEnv OCS_OPT_IPDISCOVER_MAX_ALIVE 14
	PerlSetEnv OCS_OPT_IPDISCOVER_NO_POSTPONE 0
	PerlSetEnv OCS_OPT_IPDISCOVER_USE_GROUPS 1
	PerlSetEnv OCS_OPT_GENERATE_OCS_FILES 0
	PerlSetEnv OCS_OPT_OCS_FILES_FORMAT OCS
	PerlSetEnv OCS_OPT_OCS_FILES_OVERWRITE 0
	PerlSetEnv OCS_OPT_OCS_FILES_PATH /tmp
	PerlSetEnv OCS_OPT_PROLOG_FILTER_ON 0
	PerlSetEnv OCS_OPT_INVENTORY_FILTER_ENABLED 0
	PerlSetEnv OCS_OPT_INVENTORY_FILTER_FLOOD_IP 0
	PerlSetEnv OCS_OPT_INVENTORY_FILTER_FLOOD_IP_CACHE_TIME 300
	PerlSetEnv OCS_OPT_INVENTORY_FILTER_ON 0
	PerlSetEnv OCS_OPT_DATA_FILTER 0 
	PerlSetEnv OCS_OPT_REGISTRY 1
	PerlSetEnv OCS_OPT_SNMP 0
	PerlSetEnv OCS_OPT_SNMP_INVENTORY_DIFF 1
	PerlSetEnv OCS_OPT_SNMP_PRINT_HTTPS_ERROR 1
	PerlSetEnv OCS_OPT_SESSION_VALIDITY_TIME 600
	PerlSetEnv OCS_OPT_SESSION_CLEAN_TIME 86400
	PerlSetEnv OCS_OPT_INVENTORY_SESSION_ONLY 0
	PerlSetEnv OCS_OPT_ACCEPT_TAG_UPDATE_FROM_CLIENT 0
	PerlSetEnv OCS_PLUGINS_PERL_DIR "/etc/ocsinventory-server/perl"
	PerlSetEnv OCS_PLUGINS_CONF_DIR "/etc/ocsinventory-server/plugins"
	PerlSetEnv OCS_OPT_PROXY_REVALIDATE_DELAY 3600
	PerlSetEnv OCS_OPT_UPDATE 0
	PerlModule Apache::DBI
	PerlModule Compress::Zlib
	PerlModule XML::Simple
	PerlModule Apache::Ocsinventory::Plugins::Apache
	PerlModule Apache::Ocsinventory::Plugins
	PerlModule Apache::Ocsinventory
	PerlModule Apache::Ocsinventory::Server::Constants
	PerlModule Apache::Ocsinventory::Server::System
	PerlModule Apache::Ocsinventory::Server::Communication
	PerlModule Apache::Ocsinventory::Server::Inventory
	PerlModule Apache::Ocsinventory::Server::Duplicate
	PerlModule Apache::Ocsinventory::Server::Capacities::Registry
	PerlModule Apache::Ocsinventory::Server::Capacities::Update
	PerlModule Apache::Ocsinventory::Server::Capacities::Ipdiscover
	PerlModule Apache::Ocsinventory::Server::Capacities::Download
	PerlModule Apache::Ocsinventory::Server::Capacities::Notify
	PerlModule Apache::Ocsinventory::Server::Capacities::Snmp
	<Location /ocsinventory>
		<IfModule mod_authz_core.c>
			Require all granted
		</IfModule>
		<IfModule !mod_authz_core.c>
			order deny,allow
			allow from all
		</IfModule>
		SetHandler perl-script
		PerlHandler Apache::Ocsinventory
	</Location>

	<Location /ocsplugins>
		<IfModule mod_authz_core.c>
			Require local
		</IfModule>
		<IfModule !mod_authz_core.c>
			order deny,allow
			allow from 127.0.0.1
		</IfModule>
		SetHandler perl-script
		PerlHandler Apache::Ocsinventory::Plugins::Apache
	</Location>

	PerlModule Apache::Ocsinventory::SOAP
	<location /ocsinterface>
		SetHandler perl-script
		PerlHandler "Apache::Ocsinventory::SOAP"
		<IfModule mod_authz_core.c>
			Require all granted
		</IfModule>
		<IfModule !mod_authz_core.c>
			Order deny,allow
			Allow from all
		</IfModule>
			AuthType Basic
			AuthName "OCS Inventory SOAP Area"
			AuthUserFile "APACHE_AUTH_USER_FILE"
		<IfModule mod_authz_core.c>
			Require user "SOAP_USER"
		</IfModule>
		<IfModule !mod_authz_core.c>
			require "SOAP_USER"
		</IfModule>
	</location>
</IfModule>
```

* Nota: Prestar especial atención a los parámetros de conexión con el servidor `MySQL`.

```bash
PerlSetEnv OCS_DB_HOST 127.0.0.1
PerlSetEnv OCS_DB_PORT 3306
PerlSetEnv OCS_DB_NAME ocsinventory_db
PerlSetEnv OCS_DB_LOCAL ocsinventory_db
PerlSetEnv OCS_DB_USER ocsuser
PerlSetVar OCS_DB_PWD OCSUSER_PASSWORD
```

Crear host virtual para OCSInventory-NG

```bash
nano /etc/apache2/sites-available/OCSInventory-NG.conf

<VirtualHost *:80>
    RewriteEngine on
    RewriteCond %{HTTPS} =off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [QSA,L,R=301]
</VirtualHost>

<IfModule mod_ssl.c>
	<VirtualHost inventory.dominio.cu:443>
		ServerName inventory.dominio.cu
		ServerAdmin webmaster@dominio.cu
		DocumentRoot /usr/share/ocsinventory-reports/ocsreports
		ErrorLog ${APACHE_LOG_DIR}/ocs-error.log
		CustomLog ${APACHE_LOG_DIR}/ocs-access.log combined
		SSLEngine on
		SSLCertificateFile /etc/ssl/certs/SuiteSI.crt
		SSLCertificateKeyFile /etc/ssl/private/SuiteSI.key
		<FilesMatch "\.(cgi|shtml|phtml|php)$">
			SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
			SSLOptions +StdEnvVars
		</Directory>
		BrowserMatch "MSIE [2-6]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0
		BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
		<Directory /var/lib/ocsinventory-reports/download>
		    Options Indexes FollowSymLinks MultiViews
		    <IfModule mod_authz_core.c>
				Require all granted
		    </IfModule>
		    <IfModule !mod_authz_core.c>
				Order deny,allow
				Allow from all
		    </IfModule>
		</Directory>
		Alias /download /var/lib/ocsinventory-reports/download
    </VirtualHost>
</IfModule>
```

Activar la configuración de `Apache2`, establecer persisos necesarios y reiniciar el servicio

```bash
a2enconf ocsinventory-reports.conf
a2enconf z-ocsinventory-server.conf
a2enmod ssl
a2ensite OCSInventory-NG.conf
chown www-data:www-data -R /var/lib/ocsinventory-reports
systemctl restart apache2
```

Acceder a través de un navegador web a la dirección `http://127.0.0.1/ocsreports` y configurar los parámetros de acceso a la base de datos. Concluido ese proceso, el asistente mostrará el incio de sesión, para lo cual se debe utilizar como credenciales de acceso el par usuario/contraseña `admin/admin`.

Verificar parémteros de conexión base de datos `MySQL`

```bash
cat /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
```
```php
<?php
define("DB_NAME", "ocsinventory_db");
define("SERVER_READ","127.0.0.1");
define("SERVER_WRITE","127.0.0.1");
define("COMPTE_BASE","ocsinventory");
define("PSWD_BASE","OCSUSER_PASSWORD");
?>
```

Deshabilitar el asistente web de instalación

```bash
mv /usr/share/ocsinventory-reports/ocsreports/install.php{,.org}
```

Establecer límites de subida de ficheros hasta `300Mb` en `PHP`. Es solo necesario si se fuese a utilizar `OCSInventory-NG` para el despliegue de paquetes de instalación.

```bash
sed -i "s/^upload_max_filesize = 2M/upload_max_filesize = 300M/;s/^post_max_size = 8M/post_max_size = 300M/" /etc/php/7.0/apache2/php.ini
systemctl restart apache2
```

A partir de este momento ya se está en condiciones de instalar el software agente de `OCSInventory-NG` en las estaciones y comenzar a poblar la base de datos. El despliegue del software agente se puede realizar mediante política de grupos si se tuviese un controlador de dominio, o utilizar la herramienta PsExec (sólo para Windows) y ejecutar remotamente el `Inventory Agent` con un simple comando y sus parámetros según sean necesarios. El comando puede quedar de la siguiente forma:

```bash
PsExec.exe \\NombreEquipo -u Administrador -p ADMIN_PASSWD -c OCS-NG-Windows-Agent-Setup.exe /S /NOSPLASH /NO_SYSTRAY /NOW /SSL=0 /SERVER=https://inventory.dominio.cu/ocsinventory
```

### Descargas

* [Agent para Windows 2.6.0.0 64bits](https://github.com/OCSInventory-NG/WindowsAgent/releases/download/2.6.0.0/OCS-Windows-Agent-2.6.0.0.zip)
* [Agent para Windows 2.4.0.0 32bits](https://github.com/OCSInventory-NG/WindowsAgent/releases/download/2.4.0.0/OCSNG-Windows-Agent-2.4.0.0.zip)
* [Agent para Unix/Linux 2.4.2](https://github.com/OCSInventory-NG/UnixAgent/releases/download/v2.4.2/Ocsinventory-Unix-Agent-2.4.2.tar.gz)
* [Windows Packager 2.3](https://github.com/OCSInventory-NG/Packager-for-Windows/releases/download/2.3/OCSNG-Windows-Packager-2.3.zip)
* [Agent Deployment Tool 2.3](https://github.com/OCSInventory-NG/Agent-Deployment-Tool/releases/download/2.3/OCSNG-Agent-Deploy-Tool-2.3.zip)
* [Plugins disponibles a partir de la versión 2.2](https://github.com/pluginsOCSInventory-NG)

## Breve introducción a Odoo

Odoo es un software de ERP integrado. Cuenta con una versión "comunitaria" de código abierto bajo licencia LGPLv3 y una versión empresarial bajo licencia comercial que complementa la edición comunitaria con características y servicios comerciales; ambas desarrolladas por la empresa belga Odoo S.A.

### ¿Qué es un software ERP?

Los sistemas de planificación de recursos empresariales (Enterprise Resource Planning - ERP, por sus siglas en inglés) son los sistemas de información gerenciales que integran y manejan muchos de los negocios asociados con las operaciones de producción y de los aspectos de distribución de una compañía en la producción de bienes o servicios.

## Instalación de paquetes necesarios

Existen varios métodos de instalación de Odoo, y aunque parezca paradójico, también se puede no instalar (desplegar un contenedor de Docker); todo depende del escenario. Esta guía, al estar enfocada en el uso de Debian como sistema operativo base, muestra 2 formas de hacerlo usando las herramientas instaladoras de paquetes de esta distribución.

### Servidor de bases de datos

Para su buen funcionamiento Odoo necesita un servidor PostgreSQL como gestor de bases de datos. Por defecto, la instancia de Odoo se ejecuta en el mismo host que funciona como servidor de bases de datos, aunque se pude utilizar un host distinto; pero no es objetivo de este tutorial.

```bash
apt install postgresql -y
```

En este punto, no es necesario realizar configuración alguna del servicio `postgres`. Si se quisieran establecer opciones personalizadas al servicio, es recomendable hacerlo posterior a la instalación y configuración de Odoo.

### Odoo

Tanto Odoo como la Suite SI tienen dependencias de `python`, las cuales deben ser instaladas en el sistema.

```bash
apt install libldap2-dev libsasl2-dev python-vobject python-qrcode python-pyldap python-yaml node-less python-babel python-decorator python-docutils python-feedparser python-imaging python-jinja2 python-libxslt1 python-lxml python-mako python-mock python-openid python-passlib python-psutil python-psycopg2 python-pychart python-pydot python-pyparsing python-reportlab python-requests python-suds python-vatnumber python-werkzeug python-xlwt python-pymysql python-mysql.connector python-crypto python-simplejson python-unittest2 python-ldap python-click python-beautifulsoup python-cssselect python-defusedxml python-gdata python-gevent python-gunicorn python-httplib2 python-odoorpc python-psycogreen python-pycryptopp python-pycurl python-gobject python-serial python-apt python-debian python-debianbts python-usb python-webdav python-setuptools-git python-soappy python-sqlalchemy python-zsi python-dbf python-flask python-protobuf -y
```

En el caso específico de Debian 9, deben ser instalados además los paquetes `python-support`, `python-pypdf` y `wkhtmltopdf`. Para los 2 primeros paquetes, deben usarse los disponibles en la versión de Debian 8.

```bash
wget https://packages.debian.org/jessie/all/python-support/download/python-support_1.0.15_all.deb
wget https://packages.debian.org/jessie/all/python-support/download/python-pypdf_1.13-2_all.deb
dpkg -i python-support_1.0.15_all.deb python-pypdf_1.13-2_all.deb

wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.stretch_amd64.deb
dpkg -i wkhtmltox_0.12.5-1.stretch_amd64.deb
```

La instalación del paquete `wkhtmltox` generará un error de dependecias, el cual se soluciona ejecutando:

```bash
apt install -f
```

También se deben establecer los enlaces simbólicos del paquete `wkhtmltox`.

```bash
ln -s /usr/local/bin/wkhtmltopdf /usr/bin
ln -s /usr/local/bin/wkhtmltoimage /usr/bin
```

Un vez instaladas las dependencias, se procede con Odoo.

* Primer método: utilizando repositorio de paquetes en línea de Odoo.

```bash
wget -O - https://nightly.odoo.com/odoo.key | apt-key add -
echo "deb https://nightly.odoo.com/8.0/nightly/deb/ ./" >> /etc/apt/sources.list.d/odoo.list
apt update && apt install odoo -y
```

* Segundo método: descargando la última versión del paquete.

```bash
wget https://nightly.odoo.com/8.0/nightly/deb/odoo_8.0.20171001_all.deb
dpkg -i odoo_8.0.20171001_all.deb
```

Una vez instalado Odoo usando cualquiera de los métodos propuestos, se debe asignar una contraseña al usuario `odoo` en el gestor de bases de datos y editar el fichero de configuración general del servicio.

```bash
su - postgres
psql
\password odoo
```

```bash
mv /etc/odoo/openerp-server.conf{,.org}
nano /etc/odoo/openerp-server.conf

[options]
addons_path = /usr/lib/python2.7/dist-packages/openerp/addons,/opt/suitesi/modulos
db_host = False
db_port = False
db_user = odoo
db_password = PASSWORD
db_template = template0
timezone = America/Havana
```

### Suite de Seguridad Informática

La instalación de la Suite SI se resume a la copia de los módulos que la componen `df_lt_base`, `df_lt_ssi`, `df_technical_resources_managment_no_hr`, `mail_sender_patch` y `web_datagrid`, dentro del sistema de archivos e instalarlos usando el asistente web de Odoo.

```bash
mkdir -p /opt/suite/modulos
unzip SuiteSI.zip -d /opt/suite/modulos/
chown -R root:root /opt/suite/modulos/
```

Reiniciar el servicio `odoo`.

```bash
systemctl restart odoo.service
```

Luego de copiados los módulos de la Suite SI, se accede a tavés de un navegador web a la dirección `http://suitesi.dominio.cu:8069/`, se crea la base da datos usando el asistente que se muestra y posteriormente se accede al sistema y se instalan los módulos siguiendo la ruta `Configuración\Módulos\Módulos locales`.

## Conclusiones

Aunque la Suite SI utiliza paquetería un tanto obsoleta, como son `Python v2.7` y `Odoo v8.0`; es invaluable la importancia de un proyecto de esta embargadura -dentro del sistema empresarial e incluso privado- en Cuba. Esperamos que este tutorial sirva de guía para su implementación en aquellos escenarios donde se lleve a cabo la migración de servicios a plataformas bajo software libre, apuesta hoy del país en la búsqueda de la independencia y soberanía teconológicas.

## Referencias

* [OCS Inventory Professionnel](https://www.ocsinventory-ng.org/)
* [Setting up OCS Inventory Server](https://wiki.ocsinventory-ng.org/03.Basic-documentation/Setting-up-a-OCS-Inventory-Server/)
* [OCS Inventory NG 2.5 install guide on Debian Stretch with SSL and Deployment](https://miloszengel.com/ocs-inventory-ng-2-5-install-guide-on-debian-stretch-with-ssl-and-deployment/)
* [Instalar OCS Inventory Paso a paso](http://www.latindevelopers.com/articulo/instalar-ocs-inventory/)
* [Comprehensive Perl Archive Network](https://www.cpan.org/)
* [OCS Inventory Downloads](https://ocsinventory-ng.org/?page_id=1548&lang=en)
* [ERP y CRM de código abierto | Odoo](https://www.odoo.com/es_ES/)
