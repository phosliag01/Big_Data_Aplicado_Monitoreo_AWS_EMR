# Monitoreo de un Clúster EMR con JMX Exporter, Prometheus y Grafana

## Creación del Clúster EMR

Este proyecto tiene como objetivo desplegar un sistema de monitoreo para un clúster de Amazon EMR utilizando JMX Exporter, Prometheus y Grafana. Para comenzar, es necesario acceder a la consola de AWS y crear un clúster EMR llamado EMR-Monitoring-iabdXX con las aplicaciones Hadoop, Spark y Hive. La configuración debe ser coherente con las prácticas anteriores realizadas en EMR, incluyendo un total de tres instancias: un nodo maestro y dos nodos core, además de utilizar la clave SSH predeterminada del laboratorio. Una vez lanzado, se debe esperar a que el estado del clúster sea “Waiting” para poder continuar. Luego, se realiza la conexión al nodo maestro mediante SSH utilizando el archivo .pem correspondiente y la IP pública del nodo maestro.

## Instalación y Configuración de JMX Exporter

Una vez dentro del nodo maestro, se procede a la instalación de JMX Exporter. JMX (Java Management Extensions) permite extraer métricas de procesos Java, siendo útil en este caso para acceder a estadísticas internas del NameNode de Hadoop. Se descarga el JMX Exporter desde el repositorio oficial de Maven.

A continuación, se crea un archivo de configuración llamado config.yml, donde se establecen las reglas para la recolección de métricas. En este archivo se especifica que todos los nombres y etiquetas deben estar en minúsculas y se define una regla general para capturar cualquier métrica disponible.
``` yaml
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  - pattern: ".*"
```


Para que el NameNode exponga métricas a través de JMX Exporter, se debe modificar el archivo hadoop-env.sh añadiendo una línea que incluye el agente Java con la ruta del JMX Exporter, el puerto en el que expondrá las métricas (por ejemplo, 12345) y la ruta del archivo de configuración. Una vez añadida esta línea, se reinicia el servicio del NameNode para aplicar los cambios.
```bash
sudo nano /etc/hadoop/conf/hadoop-env.sh
```
 Agrega la siguiente línea al archivo para configurar JMX Exporter
```yaml
export HADOOP_NAMENODE_OPTS="-javaagent:/home/hadoop/jmx_prometheus_javaagent-0.16.1.jar=12345:/home/hadoop/config.yml $HADOOP_NAMENODE_OPTS"
```
Reinicia el servicio del NameNode para aplicar los cambios
```bash
sudo systemctl restart hadoop-hdfs-namenode
```

## Despliegue de Prometheus y Grafana

Después de configurar el clúster EMR, creamos una instancia EC2 con Ubuntu en la misma VPC. En esta instancia se instalarán Prometheus y Grafana. 

Para Prometheus, se descarga y descomprime el archivo de instalación desde el repositorio de GitHub, y luego se ejecuta utilizando el archivo prometheus.yml como configuración. Este archivo debe ser editado para incluir como objetivo la IP del nodo maestro del clúster EMR, apuntando al puerto donde JMX Exporter está sirviendo las métricas.
Para ello usamos los comandos:
``` bash
wget https://github.com/prometheus/prometheus/releases/download/v2.30.3/prometheus-2.30.3.linux-amd64.tar.gz
tar -xzf prometheus-2.30.3.linux-amd64.tar.gz
cd prometheus-2.30.3.linux-amd64
./prometheus --config.file=prometheus.yml
```

En el archivo de configuracion de Prometheus añadimos las sguientes lineas o solo a partir de -job_name si ya existe scrape_configs. Tambien esta la opcion de sustituir lo por completo si no se quiere mantener el job de Prometheus:
``` yaml
scrape_configs:
 - job_name: 'emr-namenode'
 static_configs:
 - targets: ['172.31.30.29:12345']
```
CUIDADO: La IP hay que cambiarla por la ip del clúster EMR.

Después, se instala Grafana en la misma instancia EC2 siguiendo los pasos estándar: se agregan los repositorios necesarios, se actualiza el sistema y se instala el paquete. Una vez iniciado el servicio de Grafana, se habilita su inicio automático. Grafana se puede abrir accediendo desde un navegador a la dirección http://52.91.52.123:3000. Para conectarnos a Grafana sera necesario poner la IP publica de la maquina EC2.
```bash
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Dentro de Grafana, configuramos Prometheus como fuente de datos usando la URL http://localhost:9090. 

Finalmente, se puede crear un nuevo dashboard en Grafana que contenga paneles personalizados para visualizar métricas clave como el uso de CPU y RAM, el espacio utilizado en HDFS y el estado del NameNode, permitiendo así monitorear en tiempo real el rendimiento y estado del clúster.
