# Instalación y configuración de la Suite de Seguridad Informática en Debian 9 Stretch

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## ¿Qué es la Suite Seguridad Informática?

La Suite de Seguridad Informática (Suite SI) es un proyecto para la gestión de la seguridad a nivel empresarial, desarrollado por la división de DeSoft en Las Tunas. Esta aplicación permite la automatización de los servicios, la vigilancia y control sobre el software y hardware que conforman el patrimonio informático de una institución. A juicio de sus creadores, tiene como valor agregado promover una mayor cultura de los empresarios y funcionarios hacia la importancia de la seguridad informática. La aplicación está soportada en la versión 8 de Odoo y necesita OCSInventory para el sondeo de hardware/software de los hosts que integran la red.

```
## Breve introducción a OCSInventory NG

`Open Computer and Software Inventory Next Generation` es un software libre que permite a los Administradores de Tecnologías de la Információn gestionar el inventario de sus activos de TI. OCS-NG recopila información sobre el hardware y software de equipos existentes en la red que ejecutan el programa cliente `OCS Inventory Agent`.

## Instalación de paquetes necesarios

### Servidor de bases de datos
```

## Breve introducción a Odoo

Odoo es un software de ERP integrado. Cuenta con una versión "comunitaria" de código abierto bajo licencia LGPLv3 y una versión empresarial bajo licencia comercial que complementa la edición comunitaria con características y servicios comerciales; ambas desarrolladas por la empresa belga Odoo S.A.

### ¿Qué es un software ERP?

Los sistemas de planificación de recursos empresariales (Enterprise Resource Planning - ERP, por sus siglas en inglés) son los sistemas de información gerenciales que integran y manejan muchos de los negocios asociados con las operaciones de producción y de los aspectos de distribución de una compañía en la producción de bienes o servicios.

## Instalación de paquetes necesarios

Existen varios métodos de instalación de Odoo, y aunque parezca paradójico, también se puede no instalar (desplegar un contenedor de Docker); todo depende del escenario. Esta guía, al estar enfocada en el uso de Debian como sistema operativo base, muestra 2 formas de hacerlo usando las herramientas instaladoras de paquetes de esta distribución.

### Servidor de bases de datos

Para su buen funcionamiento Odoo necesita un servidor PostgreSQL como gestor de bases de datos. Por defecto, la instancia de Odoo se ejecuta en el mismo host que funciona como servidor de bases de datos, aunque se pude utilizar un host distinto; pero no es objetivo de este tutorial.

```
apt install postgresql -y
```

En este punto, no es necesario realizar configuración alguna del servicio `postgres`. Si se quisieran establecer opciones personalizadas al servicio, es recomendable hacerlo posterior a la instalación y configuración de Odoo.

### Odoo

Tanto Odoo como la Suite SI tienen dependencias de `python`, las cuales deben ser instaladas en el sistema.

```
apt install libldap2-dev libsasl2-dev python-vobject python-qrcode python-pyldap python-yaml node-less python-babel python-decorator python-docutils python-feedparser python-imaging python-jinja2 python-libxslt1 python-lxml python-mako python-mock python-openid python-passlib python-psutil python-psycopg2 python-pychart python-pydot python-pyparsing python-reportlab python-requests python-suds python-vatnumber python-werkzeug python-xlwt python-pymysql python-mysql.connector python-crypto python-simplejson python-unittest2 python-ldap
```

En el caso específico de Debian 9, deben ser instalados además los paquetes `python-support`, `python-pypdf` y `wkhtmltopdf`. Para los 2 primeros paquetes, deben usarse los disponibles en la versión de Debian 8.

```
wget https://packages.debian.org/jessie/all/python-support/download/python-support_1.0.15_all.deb
wget https://packages.debian.org/jessie/all/python-support/download/python-pypdf_1.13-2_all.deb
dpkg -i python-support_1.0.15_all.deb python-pypdf_1.13-2_all.deb

wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.stretch_amd64.deb
dpkg -i wkhtmltox_0.12.5-1.stretch_amd64.deb
```

La instalación del paquete `wkhtmltox` generará un error de dependecias, el cual se soluciona ejecutando:

```
apt install -f
```

También se deben establecer los enlaces simbólicos del paquete `wkhtmltox`.

```
ln -s /usr/local/bin/wkhtmltopdf /usr/bin
ln -s /usr/local/bin/wkhtmltoimage /usr/bin
```

Un vez instaladas las dependencias, se procede con Odoo.

* Primer método: utilizando repositorio de paquetes en línea de Odoo.

```
wget -O - https://nightly.odoo.com/odoo.key | apt-key add -
echo "deb https://nightly.odoo.com/8.0/nightly/deb/ ./" >> /etc/apt/sources.list.d/odoo.list
apt update && apt install odoo -y
```

* Segundo método: descargando la última versión del paquete.

```
wget https://nightly.odoo.com/8.0/nightly/deb/odoo_8.0.20171001_all.deb
dpkg -i odoo_8.0.20171001_all.deb
```

Una vez instalado Odoo usando cualquiera de los métodos propuestos, se debe asignar una contraseña al usuario `odoo` en el gestor de bases de datos y editar el fichero de configuración general del servicio.

```
su - postgres
psql
\password odoo
```

```
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

```
mkdir -p /opt/suite/modulos
unzip SuiteSI.zip -d /opt/suite/modulos/
```

Reiniciar el servicio `odoo`.

```
systemctl restart odoo.service
```

Luego de copiados los módulos de la Suite SI, se accede a tavés de un navageador web a la dirección `http://nombre_dirección_ip_servidor_odo:8069/`, se crea la base da datos usando el asistente que se muestra y posteriormente se accede al sistema y se instalan los módulos siguiendo la ruta `Configuración\Módulos\Módulos locales`.

## Conclusiones

Aunque la Suite SI utiliza paquetería un tanto obsoleta, como son `Python v2.7` y `Odoo v8.0`; es invaluable la importancia de un proyecto de esta embargadura dentro del sistema empresarial e incluso privado, en Cuba. Esperamos que este tutorial sirva de guía para su implementación en aquellos escenarios donde se lleve a cabo la migración de servicios a plataformas bajo software libre, apuesta hoy del país en la búsqueda de la independencia y soberanía teconológicas.

## Referencias

* [OCS Inventory Professionnel](https://www.ocsinventory-ng.org/)
* [ERP y CRM de código abierto | Odoo](https://www.odoo.com/es_ES/)
