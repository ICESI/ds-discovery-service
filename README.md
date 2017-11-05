### Introducción a contenedores en Linux
Universidad ICESI  
Curso: Sistemas Operativos  
Docente: Daniel Barragán C.  
Tema: Introducción al descubrimiento de servicio  
Correo: daniel.barragan at correo.icesi.edu.co

### Objetivos
* Conocer los bloques básicos para proveer descubrimiento de servicio (balanceador de carga, agentes)
* Desplegar un ambiente compuesto por microservicios y una tecnología para el descubrimiento de servicio

### Introducción
En esta guía se introducen conceptos acerca de la tecnología de descubrimiento de servicio y sus fundamentos. Además se presentan
un ejemplo de como desplegar un ambiente compuesto por microservicios y una tecnología para el descubrimiento de servicio. La información presentada es resultado de la recopilación de distintas fuentes de información, las referencias se presentan al final de la guía.

### Desarrollo
Para la realización de esta guía se requiere de 4 máquinas virtuales como muestra el diagrama realizado en clase.

![][1]
**Figura 1.** Esquema básico de descubrimiento de servicios

Se hará uso del aplicativo screen para la ejecución de comandos en background.

#### Servidor de descubrimiento de servicio
Instalar las dependencias necesarias
```
yum install -y wget unzip
wget https://releases.hashicorp.com/consul/1.0.0/consul_1.0.0_linux_amd64.zip -P /tmp
unzip /tmp/consul_1.0.0_linux_amd64.zip -d /tmp
mv /tmp/consul /usr/bin
mkdir /etc/consul.d
mkdir -p /etc/consul/data
```

Iniciar el agente en modo servidor (use una sesión de screen)
```
consul agent -server -bootstrap-expect=1 \
    -data-dir=/etc/consul/data -node=agent-server -bind=192.168.56.102 \
    -enable-script-checks=true -config-dir=/etc/consul.d -client 0.0.0.0
```

Para consultar los miembros del ambiente de descubrimiento de servicio
```
consul members
```

Para consultar los nodos/servicios en estado crítico
```
curl http://localhost:8500/v1/health/state/critical
curl http://192.168.56.102:8500/v1/health/state/critical
```

#### Microservice A
Instalar las dependencias necesarias

```
yum install -y wget unzip
wget https://bootstrap.pypa.io/get-pip.py -P /tmp
python /tmp/get-pip.py
wget https://releases.hashicorp.com/consul/1.0.0/consul_1.0.0_linux_amd64.zip -P /tmp
unzip /tmp/consul_1.0.0_linux_amd64.zip -d /tmp
mv /tmp/consul /usr/bin
mkdir /etc/consul.d
mkdir -p /etc/consul/data
```

Iniciar el microservicio (use una sesión de screen)
```
work-on microservice_a
pip install flask
vi microservice_a.py
python microservice_a.py
```

Crear un archivo de configuración para el microservicio con un  healthcheck
```
echo '{"service": {"name": "microservice_a", "tags": ["flask"], "port": 8080,
  "check": {"script": "curl localhost/health:8080 >/dev/null 2>&1", "interval": "10s"}}}' >/etc/consul.d/microservice_a.json
```

Iniciar el agente en modo cliente (use una sesión de screen)
```
consul agent -data-dir=/etc/consul/data -node=agent-one \
    -bind=192.168.56.103 -enable-script-checks=true -config-dir=/etc/consul.d
```

Una el cliente al ambiente de descubrimiento de servicio
```
consul join 192.168.56.102
```

Verifique la lista de miembros del ambiente
```
consul members
```

#### Balanceador de carga
Desde el balanceador puede realizar consulta al servidor de descubrimiento de servicio
```
curl http://192.168.56.102:8500/v1/health/state/critical
curl http://192.168.56.102:8500/v1/catalog/services
curl http://192.168.56.102:8500/v1/catalog/service/microservice_a
http://192.168.56.102:8500/v1/catalog/service/microservice_b
```

Ejemplos de respuesta
```
[{"ID":"a5646d15-ca6f-ed82-5223-cfae3733ef1b","Node":"agent-two","Address":"192.168.56.103","Datacenter":"dc1","TaggedAddresses":{"lan":"192.168.56.103","wan":"192.168.56.103"},"NodeMeta":{"consul-network-segment":""},"ServiceID":"microservice_a","ServiceName":"microservice_a","ServiceTags":["flask"],"ServiceAddress":"","ServicePort":8080,"ServiceEnableTagOverride":false,"CreateIndex":168,"ModifyIndex":168}][vagrant@localhost haproxy]
```

Instalar las dependencias necesarias
```
yum install -y wget haproxy unzip
wget https://releases.hashicorp.com/consul-template/0.19.4/consul-template_0.19.4_linux_amd64.zip -P /tmp
unzip /tmp/consul-template_0.19.4_linux_amd64.zip -d /tmp
mv /tmp/consul-template /usr/bin
mkdir /etc/consul-template
```

Configurar las plantillas de consul-template
```
vi /etc/consul-template/haproxy.tpl
```

Realice una prueba generando una vez el archivo de configuración a partir de la plantilla de formato tpl
```
consul-template -consul-addr "192.168.56.102:8500" -template "/etc/consul-template/haproxy.tpl:/etc/haproxy/haproxy.cfg" -once
```

Verifique que el archivo de configuración ha sido generado correctamente:
```
cat /etc/haproxy.cfg
```

Iniciar el agente de consul-template (use una sesión de screen)
```
consul-template -consul-addr "192.168.56.102:8500" -template "/etc/consul-template/haproxy.tpl:/etc/haproxy/haproxy.cfg:systemctl restart haproxy"
```

### Comandos Importantes
| Comando   | Descripción   |
|---|---|
| screen | Para crear una sesion de screen |
| ctrl+a, c | Para crear una nueva ventana en la misma sesión |
| ctrl+a, n | Para intercambiar de ventanas con screen |
| ctrl+a, d  | Para desligarse de una sesíón de screen |
| screen -r | Para retornar a una sesíón de screen |
| consul reload | Para recargar consul una vez se han realizado modificaciones en los archivos de configuración |

### Actividades
* Despliegue otro microservicio A de forma similar a lo realizado en la guía. Valide que el archivo de configuración de haproxy se actualiza automáticamente

* Despliegue un microservicio B de forma similar a lo realizado con el microservicio A

* Proponga una forma para el balanceo de carga del nuevo servicio

### Anexo

#### Microservice B
Instalar las dependencias necesarias

```
yum install -y wget unzip
wget https://bootstrap.pypa.io/get-pip.py -P /tmp
python /tmp/get-pip.py
wget https://releases.hashicorp.com/consul/1.0.0/consul_1.0.0_linux_amd64.zip -P /tmp
unzip /tmp/consul_1.0.0_linux_amd64.zip -d /tmp
mv /tmp/consul /usr/bin
mkdir /etc/consul.d
mkdir -p /etc/consul/data
```

Iniciar el microservicio (use una sesión de screen)
```
work-on microservice_b
pip install flask
vi microservice_b.py
python microservice_b.py
```

Crear un archivo de configuración para el microservicio con un  healthcheck
```
echo '{"service": {"name": "microservice_b", "tags": ["flask"], "port": 8080,
  "check": {"script": "curl localhost/health:8080 >/dev/null 2>&1", "interval": "10s"}}}' >/etc/consul.d/microservice_b.json
```

Iniciar el agente en modo cliente (use una sesión de screen)
```
consul agent -data-dir=/etc/consul/data -node=agent-two \
    -bind=192.168.56.104 -enable-script-checks=true -config-dir=/etc/consul.d
```

Una el cliente al ambiente de descubrimiento de servicio
```
consul join 192.168.56.102
```

Verifique la lista de miembros del ambiente
```
consul members
```

### Referencias
https://www.consul.io/docs/agent/checks.html#TTL

[1]: images/discovery_service.png
