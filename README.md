# Parcial Punto 1 - 2


# Para ejecutar la aplicación

En nuestro caso lo que se hizo fue entrar en AWS y crear una intancia donde estaria la maquina virutal de vagrant. Posteriormente se selecciona el tipo de maquina que se quiere montar y luego se escribe el nombre, con eso se crean las llaves publicas y privadas la cual permitira la conexion a la maquina virtual desde la terminal de la computadora local.


#  Instalamos Nginx

Para instalar Nginx, se debe ejecutar el siguiente comando:
```
sudo apt install nginx

```

Una vez terminen de instalarse todas las dependencias y configuraciones, al terminar la instalacion verificamos que esta instalado el servicio de Nginx con el siguiente comando:
```
sudo systemctl status nginx
```
Entramos en la carpeta de configuracion de Nginx al archivo nginx.conf y agregamos la siguiente configuarion:

```
events { }

http {
    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;

        ssl_certificate /etc/ssl/nginx/certificate.crt;
        ssl_certificate_key /etc/ssl/nginx/private.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';

        location / {
            proxy_pass http://webapp:5000;  # Cambiado "flaskapp" por "webapp"
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Claves de SSL

Utilizando el comando
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /home/vagrant/webapp/localhost.key -out /home/vagrant/webapp/localhost.crt -subj "/C=ES/ST=Valle del cauca/L=Cali/O=Universidad Autónoma de Occidente/OU=Facultad de Ingeniería/CN=Daniel Hernández Valderrama"
```
Se generan un certificado SSL y una clave dentro de la carpeta webapp y tendrán el nombre de localhost


Esto redirige todas las entradas por el puerto 80 hacia el puerto 443 y lo obliga a entrar en el protocolo seguro utilizando el certificado y la contraseña creados en el paso anterior

## Reubicación de certificado SSL y clave

Para esto solamente hay que copiar el certificado y su clave creados en webapp hacia una carpeta más adecuada, ssl/certs y ssl/private:
```
sudo cp /home/vagrant/webapp/localhost.crt /etc/ssl/certs/localhost.crt
sudo cp /home/vagrant/webapp/localhost.key /etc/ssl/private/localhost.key
```


#  Verifica que Docker y docker-compose 

Para verificar que funciona el docker y docker-compose, es necesario colocar las siguientes líneas:
```

sudo docker-compose up --build
```

## 

#  Prometheus +  Node Exporter

Para realizar este punto usando una maquina virtual en local por la facilidad de implementar los servicios
que contienen estos 2 aplicativos, Prometheus será desplegado y posteriormente se configurara para que
nos permitira observar las tareas que se estan ejecutando, siendo uno de ellos el servicio
de node-exporter que esta exportando la información (La maquina de Vagrant),
esta información contiene mediciones tales como el uso de la CPU, RAM Disponible y Disco duro.

Archivo prometheus.yml
```
global:
  scrape_interval:     15s 
  evaluation_interval: 15s 

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['prometheus:9090']

  - job_name: 'node_exporter'
    static_configs:
    - targets: ['node_exporter:9100']
```
El archivo esta defiendo el intervalo en el cual se scrapea la información de las tareas
y también el intervalo en el cual se evaluan las alertas, en este caso contamos
con 2 archivos de configuraciones teniendo uno la información del propio prometheus
que normalmente estaríamos hablando de información de su entorno  y en otra
configuración va la configuración al scrapear el node_exporter con la información del nodo.

```
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: unless-stopped
    volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
    - prometheus-data:/prometheus
    ports:
    - 9090:9090
    command:
    - '--config.file=/etc/prometheus/prometheus.yml'
    - '--storage.tsdb.path=/prometheus'
    - '--storage.tsdb.retention.time=1y'
    - '--web.enable-lifecycle'

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    restart: unless-stopped
    pid: host
    ports:
    - 9100:9100
    command:
      - '--path.rootfs=/host'
    volumes:
     - '/:/host:ro,rslave'
```

Para el node-exporter es importante notar que el PID se ejecutara cómo HOST y los volumenes
los cuales poseera sera la información de la maquina entera, esto es importante porque si no fuera así
la información que podría proveer el node-exporter sería la del contenedor y no de la maquina host.

Es importante aclarar que si se instala el Prometheus sin el node-exporter, el prometheus no podrá recolectar
información de la maquina host, por lo cual es importante tener en cuenta que si se quiere recolectar información
de la maquina host se debe instalar el node-exporter o ya sea primero o al mismo tiempo.

# Punto 4 - Grafana con Prometheus y Node Exporter
Para grafana a si mismo se levanto el servicio, el cual esta exponiendo el puerto 3000
con credenciales admin:admin que nos dejara entrar y configurar los paneles, el data source de prometheus
y otras herramientas cómo los puntos de contacto.

Para acceder al Grafana podemos dirigirnos a 192.168.50.4:3000 para visualizar el aplicativo web y entrar
cómo usuario: admin y contraseña: admin, aquí podremos ver que el grafana no tendra ningun tipo de informacion, a lo cual
nos tocará dirigirnos al apartado de Conexiones y añadir una nueva Fuente de Datos cómo lo es prometheus
y poner la dirección URL: http://192.168.50.4:9090 con esto ya podremos ir a Dashboards y crear nuestros
dashboards personalizados.

Por si desea tener un dashboard preconfigurado con el cual ver información de su nodo, le recomendamos importar
el dashboard que esta en esta misma carpeta con el nombre "1860_rev37.json", así mismo si un día gustaría de compartir
un panel a un compañero podría hacerlo exportando su modelo.json para que este pueda importarlo
en su grafana.

