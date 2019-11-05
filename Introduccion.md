# Instalación y configuración del servidor bind9 en nuestra red local

## Se configura el fichero /etc/bind9/named.conf.local:
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

## Declaración de la zona directa
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


## Declaración de la zona inversa
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




