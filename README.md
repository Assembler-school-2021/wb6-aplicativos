# wb6-aplicativos

> Pregunta 1 : Cómo puedo verificar que el registro está ya activo y funcionando usando herramientas de consola?
Hay que instalar dig con:
```
apt install dnsutils
```
Con dig haciendo consulta sobre un registro de tipo A:
```
dig -t A wp.devops-alumno08.com
```

> Pregunta 2 : Que ha ocurrido? Amplia los límites de upload de php para permitir hasta 32mb.
	
Cambiamos en el php.ini de */etc/php7.3/fpm/* los limites de las variables:
```
post_max_size = 32M
upload_max_filesize = 32M
```
	
> Pregunta 3 : Por defecto php viene bastante capado. Busca una forma de augmentar el pool máximo de workers a 25, con un mínimo de 10.

Editamos el fichero */etc/php/7.3/fpm/pool.d/www.conf* y añadimos estos valores
```
pm = dynamic
pm.max_children = 25
pm.start_servers = 10
pm.min_spare_servers = 10
pm.max_spare_servers = 25
```

> Pregunga 4 : Configura OPcache correctamente.
Modificamos el fichero */etc/php/7.3/fpm/conf.d/10-opcache.ini*
```
[opcache]
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
opcache.enable_cli=1
```

## Wordpress - Apache
Crea un subdominio diferente. A continuación despliega otro wordpress distinto en ese dominio pero usando apache con php-fpm. Deberas parar nginx primero antes de instalar apache.

Busca la documentación necesaria en internet. Puedes usar busquedas cómo "debian buster apache wordpress" .

> Pregunta 5 : Documenta todo el proceso.

Usamos esta guía:

https://chachocool.com/como-instalar-wordpress-en-debian-10-buster/

`service nginx stop`

Instalar libapache2-mod-php7.3
	
`apt install libapache2-mod-php7.3`
  
Instalar el certbot
```
	apt install python-certbot-apache
	certbot --apache
```
Esta es la config que tenemos que tener:
```
	<IfModule mod_ssl.c>
	  <VirtualHost *:443>
	    ServerName wp2.devops-alumno08.com
	    DocumentRoot /var/www/wordpress2
	    AddType application/x-httpd-php .php
	
	    <Directory /var/www/wordpress2>
	          AllowOverride all
	    </Directory>
	    <FilesMatch \.php$>
	        SetHandler application/x-httpd-php
	    </FilesMatch>
	
	    SSLCertificateFile /etc/letsencrypt/live/wp2.devops-alumno08.com/fullchain.pem
	    SSLCertificateKeyFile /etc/letsencrypt/live/wp2.devops-alumno08.com/privkey.pem
	    Include /etc/letsencrypt/options-ssl-apache.conf
	  </VirtualHost>
  </IfModule>
```

## Prestashop
De nuevo crea otro subdominio para publicar una tienda online. Esta vez usa la documentación oficial usando nginx y php-fpm. Deberas parar apache antes de arrancar nginx.

https://devdocs.prestashop.com/1.7/basics/installation/

> Pregunta 6 : Documenta todo el proceso.

Instalamos composer:
```
	wget https://getcomposer.org/download/1.10.17/composer.phar -O /usr/bin/composer
	chmod +x /usr/bin/composer
```

Seguimos los pasos de esta guía oficial:
https://devdocs.prestashop.com/1.7/basics/installation/localhost/

> Pregunta 7 : Como podemos separar los pools de fpm? Crea 3 pools independientes, cada uno para su aplicativo con un socket independiente. Usaremos los 3 anteriores.

En el directorio */etc/php/7.3/fpm/pool.d/* podemos crear tantas pools como deseemos simplemente cambiando el nombre de archivo, de la pool y del socket. Para prestashop hemos creado un pool llamado www-ps (ver fichero www-ps.conf) con estos valores:
```
[www-ps]
listen = /run/php/php7.3-fpm-ps.sock
```

Y para wordpress hemos creado este (ver fichero www-wp.conf):
```
[www-wp]
listen = /run/php/php7.3-fpm-wp.sock
```

Reniciamos el servicio php y vemos cómo tenemos disponibles los diferentes pools creados:

![image](https://user-images.githubusercontent.com/65896169/129960726-5d8e456d-545c-4b1a-b40a-82fff60b962a.png)

> Pregunta 8 : Configura tanto nginx como php para que publiquen sus estadisticas, y que solo sean accesibles desde 127.0.0.1 . En concreto de nginx queremos el stub_status y de fpm el pm.status.

Para publicar las estadísticas de nginx, solamente hay incluir la siguiente location en el server por defecto de nginx (en */etc/nginx/sites-enabled/default*):
```

        location /nginx_status {
                stub_status;
                allow 127.0.0.1;        #only allow requests from localhost
                deny all;               #deny all other hosts
        }
```

Reininciamos el servicio y ya podemos acceder desde localhost:
```
# service nginx restart
# curl localhost/nginx_status
Active connections: 1
server accepts handled requests
 1 1 1
Reading: 0 Writing: 1 Waiting: 0
```

Y para php podemos crear un pool nuevo que habilite las estadísticas (*/etc/php/7.3/fpm/pools.d/stats.conf*), aunque lo lógico sería habilitar en cada pool que tenemos, ya que solo nos ofrece la estadisticas del pool en cuestión:
```
[stats]
user = www-data
group = www-data
listen = /run/php/php7.3-fpm-stats.sock

listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 1
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 1

pm.status_path = /status
```
Y en el sitio por defecto de nginx añadimos esta location:
```
        location ~ ^/(status|ping)$ {
                allow 127.0.0.1;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_pass   unix:/run/php/php7.3-fpm-stats.sock;
        }
```

Reinciamos los servicios de nginx y php-fpm y ya podemos acceder a la información en localhost:

```
# curl localhost/status
pool:                 stats
process manager:      dynamic
start time:           18/Aug/2021:21:59:15 +0200
start since:          227
accepted conn:        1
listen queue:         0
max listen queue:     0
listen queue len:     0
idle processes:       0
active processes:     1
total processes:      1
max active processes: 1
max children reached: 0
slow requests:        0
```
