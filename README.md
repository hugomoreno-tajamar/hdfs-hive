
## Requisitos previos

Para poder utilizar esta configuración es necesario contar con:

- [Docker](https://www.docker.com/) instalado en tu máquina.
- Una terminal de wsl para windows o una máquina linux

## Creación del clúster e inicialización

Para crear el clúster, nos vamos a ayudar de un proyecto de github:
- https://github.com/big-data-europe/docker-hive/blob/master/

Ahora, vamos a hacer un git clone de este repositorio para tener todos los archivos

```bash
git clone https://github.com/big-data-europe/docker-hive/blob/master/
```

Para añadir más detalle, vamos a modificar el docker-compose para añadir yarn y map-reduce en nuestro clúster


```bash
version: "3"

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop2.7.4-java8
    volumes:
      - namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop-hive.env
    ports:
      - "50070:50070"

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    volumes:
      - datanode:/hadoop/dfs/data
    env_file:
      - ./hadoop-hive.env
    environment:
      SERVICE_PRECONDITION: "namenode:50070"
    ports:
      - "50075:50075"

  resourcemanager:
    image: bde2020/hadoop-resourcemanager:2.0.0-hadoop2.7.4-java8
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
      - YARN_RESOURCEMANAGER_HOSTNAME=resourcemanager
    env_file:
      - ./hadoop-hive.env
    depends_on:
      - namenode
    ports:
      - "8088:8088" # Puerto para la interfaz web de YARN ResourceManager

  nodemanager:
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop2.7.4-java8
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
      - YARN_NODEMANAGER_HOSTNAME=nodemanager
    env_file:
      - ./hadoop-hive.env
    depends_on:
      - resourcemanager

  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      SERVICE_PRECONDITION: "hive-metastore:9083"
    ports:
      - "10000:10000"

  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    command: /opt/hive/bin/hive --service metastore
    environment:
      SERVICE_PRECONDITION: "namenode:50070 datanode:50075 hive-metastore-postgresql:5432"
    ports:
      - "9083:9083"

  hive-metastore-postgresql:
    image: bde2020/hive-metastore-postgresql:2.3.0

  presto-coordinator:
    image: shawnzhu/prestodb:0.181
    ports:
      - "8080:8080"

volumes:
  namenode:
  datanode:

```

# Trabajando con Hive

Ahora, accedemos a la shell de hive ejecutando el contenedor

```bash
docker exec -it docker-hive-hive-server-1 bash
```
Una vez estamos dentro, entramos en el directorio hive y escribimos hive para entrar al servicio de hive

```bash
cd hive
hive
```
Ya estamos dentro del servicio de hive, vamos a probar a hacer una consulta.

Para ello, debemos de seguir estos pasos:

1.- Crear una base de datos

```bash
CREATE DATABASE ejemplo;
```

2.- Crear una tabla, pero primero debemos de especificar que queremos usar la base de datos ejemplo

```bash
USE ejemplo;
```
```bash
CREATE TABLE test_table (
    id INT,
    name STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

3.- Insertamos datos en la base de datos
```bash
INSERT INTO test_table VALUES (1, 'Alice'), (2, 'Bob');
```
4.- Hacemos una consulta simple a la tabla
```bash
SELECT * FROM test_table;
```

Por último, Hive almacena los datos en HDFS, y puedes verificar los archivos de la tabla en el directorio de almacenamiento configurado (por defecto suele ser /user/hive/warehouse).

Por lo que, saliendo del servicio de hive, podemos ver el directorio:

```bash
hdfs dfs -ls /user/hive/warehouse/
```

Y veremos que el archivo que hay dentro contiene la información de nuestra tabla

```bash
hdfs dfs -cat /user/hive/warehouse/<nombre_archivo>
```

1