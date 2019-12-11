# Instalación y configuración del servidor bind9 en nuestra red local

## Configuración de las zonas
### Configuración del fichero /etc/bind9/named.conf.local:
~~~
zone "iesgn.org"
{
	file "db.iesgn.org";
	type "master";
};
zone "0.0.10.in-addr.arpa"
{
	file "db.10.0.0";
type "master";
};
~~~
Existen dos tipos de servidores, maestos y esclavos, en nuestro caso, usamos el tipo maestro porque solo tenemos un servidor.

Se declaran las dos zonas que van a haber, la zona directa y la inversa. 

### Declaración de la zona directa
El fichero de configuración se debe crear en /var/cache/bind/db.iesgn.org donde se ponen los registros:
~~~
@	in	SOA	dns.iesgn.org.	palomagarciacampon08@.gmail.com
			serial 1;
			(4 tiempos)
@	in	NS	dns.iesgn.org.
$ORIGIN			iesgn.org.
dns	in	A	10.0.0.3
web	in	A	10.0.0.2
cliente1 in	A	10.0.0.4
cliente2 in	A	10.0.0.5
correo	in	A	10.0.0.200
www	in	CNAME	web
departamentos in CNAME	web
ftp	in	CNAME	web
@	in	MX	10 correo
~~~
El SOA guarda información de la zona, y hay que guardar 4 cosas:
- El nombre del servidor con autoridad maestro. En este caso solo hay uno, así que  está claro que hay que poner.
- Correo electrónico de la persona responsable.
- Serial, que son un número.
- Y 4 tiempos en segundos y tienen que ver con los maestros y los esclavos.

NS, el nombre del servidor con autoridad en la zona. 


### Declaración de la zona inversa
Se configura /var/cavhe/bind/db.10.0.0
~~~
@	in	SOA	dns.iesgn.org.	palomagarciacampon08@.gmail.com
			serial 1;
			(4 tiempos)
@	in	NS	dns.iesgn.org.
$ORIGIN 0.0.10.in-addr.arpa.
3	in	PTR	dns.iesgn.org.
2	in	PTR	dns.iesgn.org.
4	in	PTR	dns.iesgn.org.
5	in	PTR	dns.iesgn.org.
200	in	PTR	dns.iesgn.org.
~~~

El registro SOA de la zona inversa es igual que en la zona directa.



## Servidor maestro/esclavo
Cada cierto tiempo, las zonas que tienen maestro se sincronizan, por lo que si el maestro falla responde un esclavo. Pero los esclavos no se tocan, entre ellos se configuran.

Un servidor esclavo contiene una réplica de las zonas del maestro:
- DNS maestro: dns-1.ejemplo.com (10.0.0.10)
- DNS esclavo: dns-2.ejemplo.com (10.0.0.5)

Se debe producir una transferecia de zona (el esclavo hace una solicutus de la zona completa del maestro para que se sincronicen lo servidores. 

Por seguridad, sólo deben aceptar transferencias de zonas hacia los esclavos autorizados, para ello en el fichero /etc/bind/named.conf.options, deshabilitamos la tranferencia:
~~~
options{
	...
	allow-transfer { none; }
}
~~~
Por defecto no se transfiere a nadie. PEro hay otras formas de configuración:

~~~
zone ejemplo.com {
	type master;
	file "db.ejemplo.com";
	allow-transfer { 10.0.0.5; }
};
~~~

~~~
zone ejemplo.com {
	type slave;
	file "db.ejemplo.com";
	allow-transfer { 10.0.0.10; }
};
~~~

### Zona directa del maestro
~~~
@	in	ns		dns-2.ejemplo.com.
...
dns-2	in	A 		10.0.0.5
~~~

### Transferencia de zona
Cuando se reinicia el servidor esclavo podemos ver como se ha producido una transferencia de zona:
~~~
systemctl restart bind9
tail /var/log/syslog
~~~

En syslog se puede ver si se hace correctamente la transferencia.


### ¿Cuándo se hacen las copias?
El esclavo solo iniciará la copia cuando el número de serie configurado en la SOA aumente.
~~~
$TTS 86400
@	in 	SOA	dns-1.ejemplo.com.	root.ejemplo.com. (
		# este número es el que hay que aumentar 
		# para que los esclavos se actualizan
		1	; serial
		604800	; refresh
		86400	; retry
		2419200	; expire
		86400)	; negativa cache TTL
...
~~~

Pero puede pasar que se cambie el maestro cambiiando el número de serie pero al esclavo no le llegue. ESte fallo no se notifica.

El formato recomendado del número de serie es YYMMDDNN: 19111801 (año, mes, dia, número).

**Primer tiempo**
Como puede haber algunas pérdidas de paquetes, el esclavo le pregunta al maestro en un intervalo de de tiempo (refresh). 

**Segundo tiempo**
retry: si el esclavo pregunta pero este no responde lo va a intentar cada cierto tiempo más pequeño que el refresh.

**Tercer tiempo**
expire: tras el tiempo transcurrido, el maestro no responde al retry. Durente este tiempo, el esclavo ha seguido con la zona del esclavo, pero si se cumple el tiempo y no responde se inmola. 

**Cuarto tiempo**
Cuando pregunta por un nombre que no existe se guarda en caché durante este tiempo.


## Evitar errores
- cada vez que se realice una modificación recuerda incrementar el número de serie.
- Para detectar errores de sintaxis puedes usar el comando:
~~~
named-checkzone ejamplo.com...
~~~

Solicitar una zona completa desde el esclavo al maestro:
~~~
dig @x.x.x.x ejemplo.com. axfr
~~~


## Subdominios en bind9
### Subdominios virtuales
www.ejemplo.com {www.es.ejemplo.com, www.doc.ejemplo.com}
1 dns tiene autoridad sobre todo.

Este no es el tipo de subdominio que usaremos en las prácticas.

~~~
$ORIGIN es.ejempo.com
web	in	A 	10.0.0.100
www	in	CNAME	web
~~~


### Delegación de subdominios
1 dns tiene dominio sobre www.ejemplo.com
1 dns tiene autoridad sobre www.es.ejemplo.com

Cuando a un servidor DNS le pregunten por el subdominio le redirege la pregunta (realmente no redirige sino que pregunta de nuevo) al servidor DNS que controla el subdominio. 
~~~
$ORIGIN	es.ejemplo.com.
@	in	ns	dns-3
dns-3	in	A	10.0.0.13
~~~

