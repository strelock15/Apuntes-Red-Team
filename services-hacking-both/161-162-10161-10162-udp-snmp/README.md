---
description: Explotar el servicio SNMP.
---

# 161,162,10161,10162/udp - SNMP

## INFORMACIÓN BÁSICA

El **Protocolo simple de administración de red** es un protocolo de la **capa de aplicación** que facilita el intercambio de información de administración entre dispositivos de red. Los dispositivos que normalmente soportan SNMP incluyen routers, switches, servidores, estaciones de trabajo, impresoras, bastidores de módem y muchos más. Permite a los administradores supervisar el funcionamiento de la red, buscar y resolver sus problemas, y planear su crecimiento.

Se basa en un gerente que centraliza la información y varios agentes.\
Utilza los siguientes puertos por defecto:

**UDP 161**

**UDP 162 (TRAP)** → Recepción de los paquetes del servidor.

También tiene puertos por defecto si queremos trabajar en cifrado:

**UDP 10161 (TLS)**

**UDP 10162 (TRAP TLS)**

La **version** mas actualizada de SNMP actualmente es **SNMP3**.

Se basa en un **GERENTE** que centraliza la información y varios **AGENTES** que envían la información y ejecutan las ordenes del gerente.

Para que se puede recibir la información de manera ordenada se utilizan **MIB (BASE DE INFORMACIÓN PARA LA GESTIÓN)**

### MIB

Es una colección de información organizada jerarquicamente a la que se accede mediante el protocolo SNMP.&#x20;

Hay dos tipos de MIBS:

* Escalar: define una única instancia de objeto.
* Tabular: Define varias instancias de objeto interconectadas y agrupadas en tablas.

### OIDs

Son Identificadores de Objetos. Únicamente identifican objetos en la jerarquía MIB. Puede ser dibujado como un arbol en los que los niveles se asignan a diferentes organizaciones.

Las empresas que ofrecen este servicio definen ramas privadas que incluyen objetos para sus propios productos.

![OID ejemplo.](../../.gitbook/assets/snmp1.png)

Puedes navegar a través de un OID desde la siguiente página:

{% embed url="http://www.oid-info.com/cgi-bin/display?tree=#focus" %}
OID ejemplo
{% endembed %}

O ver como se define un OID en la siguiente página:

{% embed url="http://oid-info.com/get/1.3.6.1.2.1.1" %}

Existen OIDs bien conocidos como los siguientes:

* **1.3.6.1.4.1.77.1.2.25** → enumera usuarios
* **1.3.6.1.2.1.25.4.2.1.2** → enumera procesos de Windows
* **1.3.6.1.2.1.6.13.1.3** → enumera puertos abiertos
* **1.3.6.1.2.1.25.6.3.1.2** → enumera software instalado con version y todo.

### Ejemplo OID

{% embed url="https://www.netadmintools.com/snmp-mib-and-oids" %}
Fuente
{% endembed %}

**`1 . 3 . 6 . 1 . 4 . 1 . 1452 . 1 . 2 . 5 . 1 . 3. 21 . 1 . 4 . 7`**

* **1 -** Se denomina ISO y es lo que establece que nos encontramos ante un OID. Por eso todos empiezan por 1.
* **3 -** Se denomina ORG e indica la organización que construyó el equipo.
* **6 -** es el DoD o Departamento de defensa y es quien construyó internet.
* **1 -** Es el valor de Internet e indica que las comunicaciones se realizan por internet.
* **4 -** Este indica que la máquina no es del gobireno sino de una organización privada.
* **1 -** Este indica que la máquina está hecha por una entidad comercial.

Estos primeros 6 valores suelen ser siempre iguales y nos dan la información básica de los OIDs.&#x20;

Los siguientes números son los importantes:

* **1452 -** Es el nombre de la organización fabricante del equipo.
* **1 -** Es el tipo de equipo. En este caso es un reloj con alarma.
* **2 -** Determina que este equipo es un terminal remoto.

Los demás números dan información concreta sobre el equipo (no los voy a traducir):

* **5 –** denotes a discrete alarm point.
* **1 –** specific point in the device
* **3 –** port
* **21 –** address of the port
* **1 –** display for the port
* **4 –** point number
* **7 –** state of the point

### Versiones SNMP

Hay tres versiones importantes:

* SNMPv1: Es la principal. Sigue siendo la más frecuente. La autenticación se basa en una cadena de texto (community string) que viaja en texto plano (igual que el resto de la información).&#x20;
* SNMPv2, SNMPv2c: No existen cambios sustanciales en cuanto a seguridad.
* SNMPv3: Usa una forma de autenticación mejorada y la información viaja encriptada. Se podría realizar un ataque de diccionario pero sería más complejo.

### Community Strings

Como hemos dicho antes, para acceder a la información de un **MIB** necesitas conocer la **community string** en las versiones 1 y 2 / 2c y los credenciales en la versión 3.

Hay dos tipos de **Community strings**:

* **public**: con funciones de solo lectura
* **private**: normalmente se puede leer y escribir en ella.

{% hint style="info" %}
IMPORTANTE

La capacidad de escritura de un OID depende de la Community string utilizada. Existen algunas "public" que pueden ser escritas y, en cambio, puede haber objetos que siempre sean de solo lectura.

Si intentas escribir un objeto se obtendrá un erro de **noSuchName** o **readOnly**.

En las versiones 1 y 2, si se utilizaba una Community String incorrecta, el servidor no contestaba por lo que **si el servidor contesta, tenemos una Community string válida.**.
{% endhint %}

## FUERZA BRUTA A LA COMMUNITY STRING

Para adivinar la Community String (SNMPv1/v2), puedes realizar un ataque de diccionario:

{% embed url="https://app.gitbook.com/s/-MhLaK1R01rLyFh-Jcoj/techniques/fuerza-bruta/cheatsheet#snmp" %}
Artículo sobre el tema.
{% endembed %}

## ENUMERACIÓN SNMP

Es recomendable instalar el siguiente programa para saber qué significa cada OID del dispositivo:

```
apt-get install snmp-mibs-downloader
download-mibs
```

Si tienes una **Community String válida** puedes acceder a los datos utilizando **SNMPWalk** o **SNMP-Check**:

```
snmpwalk -v [VERSION_SNMP] -c [COMM_STRING] [DIR_IP]
snmpwalk -v [VERSION_SNMP] -c [COMM_STRING] [DIR_IP] 1.3.6.1.2.1.4.34.1.3 #Get IPv6, needed dec2hex
snmpwalk -v [VERSION_SNMP] -c [COMM_STRING] [DIR_IP] NET-SNMP-EXTEND-MIB::nsExtendObjects #get extended
snmpwalk -v [VERSION_SNMP] -c [COMM_STRING] [DIR_IP] .1 #Enum all
snmp-check [DIR_IP] -p [PORT] -c [COMM_STRING]
nmap --script "snmp* and not snmp-brute" <target>
```

Gracias a las busquedas extendidas (obtenidas al instalar mibs) podemos enumerar más información sobre el sistema:

```
snmpwalk -v X -c public <IP> NET-SNMP-EXTEND-MIB::nsExtendOutputFull
```

SNMP tiene mucha información sobre el Host. Algunas cosas muy interesantes son:

* Interfaces de red
* Nombres de usuario
* Tiempo activa
* Versión del S.O.
* Procesos activos.

## MASSIVE SNMP

{% embed url="https://github.com/mteg/braa" %}
Braa
{% endembed %}

Braa es un escaner masivo de SNMP. El uso de esta herramienta es, sin duda, realizar busquedas SNMP. Sin embargo, es capaz de realizar docenas o centenares de busquedas en diferentes hosts en un solo proceso.

**Syntax:** braa \[Community-string]@\[IP of SNMP server]:\[iso id]

```
braa ignite123@192.168.1.125:.1.3.6.*
```

Esto puede obtener muchos MB de información que no se pueden analizar a mano. Así que con el siguiente artículo podremos averiguar que información es interesante:

{% embed url="https://www.rapid7.com/blog/post/2016/05/05/snmp-data-harvesting-during-penetration-testing" %}
Fuente
{% endembed %}

### Dispositivos

Lo primero que debemos hacer es extraer el sysDesc `.1.3.6.1.2.1.1.1.0 MIB` de cada archivo para determinar de qué equipos he obtenido información:

```
grep ".1.3.6.1.2.1.1.1.0" *.snmp
```

### Identificar la private string

Como ejemplo, si puedo identificar la Community String que una organización utiliza en su "**Cisco IOS router**", entonces puedo obtener la configuración de dichos routers.

La mejor forma de obtener esos datos es con SNMP trap data:

```
grep -i "trap" *.snmp
```

### Nombre de usuario / contraseña

Otra area interesante son los logs. Hay equipos que guardan logs en sus tablas MIB. Estos logs pueden contener intentos fallidos de auteticación.&#x20;

Esto en un entorno real puede acercarnos a los credenciales correctos si alguien ha introducido por error parte de su contraseña en el nombre de usuario (que puede pasar).

```
grep -i "login\|fail" *.snmp
```

### Emails

```
grep -E -o "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b" *.snmp
```

## Modificar valores SNMP

Puedes utilizar **NetScanTools** para modificar valores. Para ello necesitas la **private** **string**.

## SPOOFING

Si tenemos un ACL que solo permite a determinadas IPs realizar busquedas en el servicio SNMP, se puede spoofear una de dichas direcciones dentro de un paquete UDP esnifado.

## ARCHIVOS DE CONFIGURACIÓN

* snmp.conf
* snmpd.conf
* snmp-config.xml
