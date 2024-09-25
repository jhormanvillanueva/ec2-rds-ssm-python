# Configurar una VPC para implementar un servidor web y una base de datos en AWS

En esta guía, configuraré una nube privada virtual (VPC) para implementar un servidor web, escrito en Python, en una instancia de Amazon EC2 dentro de una subred pública y una base de datos MySQL en AWS RDS dentro de dos subredes privadas.

![Flask](https://img.shields.io/badge/flask-%23000.svg?style=for-the-badge&logo=flask&logoColor=white) ![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

## Voy a implementar la próxima arquitectura de nube en AWS
![arquitectura aws](img/EC2-RDS.svg)

<hr>
1. Construiré la arquitectura de red utilizando el servicio AWS VPC.

- Voy a crear una VPC con el siguiente bloque CIDR IPv4
   - 192.168.0.0/16

- Voy a crear las tres subredes con los siguientes nombres:
   - **PublicSubnetA**:
      - Zona de disponibilidad: a
      - CIDR: 192.168.1.0/24

   - **PrivateSubnetA**:
      - Zona de disponibilidad: a
      - CIDR: 192.168.2.0/24

   - **PrivateSubnetB**:
      - Zona de disponibilidad: b
      - CIDR: 192.168.3.0/24

Necesito que la VPC tenga una conexión a Internet, por lo que debo configurar un **Internet Gateway**.
- Cuando se crea el Internet Gateway, lo relaciono a la VPC.
- Creo una Tabla de enrutamiento (Route Table)
- En la Tabla de enrutamiento (Route Table) le asocio la **Subred Pública** y creo una ruta para permitir la conexión a Internet a través del **Internet Gateway**

<hr>
2. Utilizaré el servicio AWS System Manager para almacenar los parámetros de conexión que utilizará el servidor web para conectarse a la base de datos configurada en AWS RDS.

   - Utilizando AWS System Manager y en la opción Parameter Store creo los siguientes parámetros:
      - /book/user: root
      - /book/password: *Test!2024* utilizando el tipo *SecureString*
      - /book/database: books_db
      - /book/host: 192.168.1.23 *La dirección IP donde se encuentra la base de datos. Al crear la base de datos en RDS necesito cambiar esta dirección IP al endpoint proporcionado por RDS*.

- El servidor web se ejecutará en la instancia EC2 y necesita leer los parámetros de conexión a la base de datos implementada en RDS, por lo que necesito crear un rol de IAM que tenga permiso para que EC2 lea los parámetros de conexión desde el servicio System Manager.
   - En el servicio IAM, creo un rol llamado *ec2RoleSSM*.
   - El rol tiene el siguiente permiso:
      - SSMFullAccess

<hr>

3. Voy al servicio AWS EC2 y creo dos grupos de seguridad.

- El primer grupo de seguridad tiene los siguientes parámetros:

   - Nombre: web-server-SG
   - Reglas de entrada:
      - Tipo: TCP personalizado
      - Rango de puertos: 5000
      - Origen: Cualquier lugar

- El segundo grupo de seguridad tiene los siguientes parámetros:

   - Nombre: database-SG
   - Reglas de entrada:
      - Tipo: MYSQL/Aurora
      - Rango de puertos: 3306
      - Origen: web-server-SG *El SG del servidor web*
<hr>

4. Voy al servicio AWS EC2 y lanzo una instancia EC2 en la **PublicSubnet** con las siguientes configuraciones
   - AMI: *Amazon Linux 2023*
   - Tipo de instancia: *t2.micro*
   - Par de claves: asociar un par de claves
   - Configuración de red:
      - VPC
      - Subred pública: habilitar *IP pública*
      - Asociar el grupo de seguridad del servidor web
   - Detalles avanzados:
      - Perfil de instancia de IAM: asociar el rol creado anteriormente
      - Datos del usuario: *copia las siguientes líneas a los datos del usuario*
         ```
         #!/bin/bash
         sudo dnf install -y python3.9-pip
         pip install virtualenv
         sudo dnf install -y mariadb105-server
         sudo service mariadb start
         sudo chkconfig mariadb on
         pip install flask
         pip install mysql-connector-python
         pip install boto3
         ```
   - Cuando finaliza el lanzamiento de la instancia, me conecto a la terminal y clono este repositorio usando la URL correspondiente:

         
         git clone -URL-
         

   - Para ejecutar el servidor web, ejecuto el siguiente comando en el directorio donde se encuentra app.py. Debe asegurarse que el grupo de seguridad tenga habilitado el puerto apropiado.

         python3 -m virtualenv venv
         source venv/bin/activate
         python app.py

   - La base de datos no está configurada. Voy al directorio llamado db y ejecuto los siguientes comandos

         sudo chmod +x set-root-user.sh createdb.sh
         sudo ./set-root-user.sh
         sudo ./createdb.sh
   - Puedes comprobar si la base de datos se ha creado ejecutando el siguiente comando:

         sudo mysql
         show databases;
         use books_db;
         show tables;
         SELECT * FROM Books;
   - La base de datos no está configurada en AWS RDS sino en AWS EC2. Por lo tanto, en el siguiente paso, voy a configurar AWS RDS.

<hr>

5. Voy a AWS RDS para configurar el servicio de base de datos relacional.

- Creo un grupo de subredes para privateSubnetA y privateSubnetB.
- Creo AWS RDS con los siguientes parámetros:

   - Tipo de motor: MariaDB o MySQL
   - Plantillas: nivel gratuito

      - Nombre de usuario maestro: el mismo usuario que creó en AWS System Manager
      - Contraseña maestra: la misma contraseña que creó en AWS System Manager.
      - Nube privada virtual (VPC): la VPC que se creó en el paso uno.
      - Grupo de subredes de base de datos: el grupo de subredes de base de datos creado.
      - Grupos de seguridad de VPC existentes: Asocio el grupo de seguridad de base de datos creado en el paso tres.

- Cuando finalmente se crea AWS RDS, copio el punto de conexión de RDS. Puede actualizar el parámetro /book/host en AWS System Manager.
<hr>

7. En este paso, migraré la base de datos de AWS EC2 a AWS RDS.

- Desde la terminal de la instancia de AWS EC2, ejecuto los siguientes comandos:
- Verifico la conexión a AWS RDS desde AWS EC2

      mysql -u root -p --host rds-endpoint
      show databases;

- Comienzo con la migración con los siguientes comandos:

      mysqldump --databases books_db -u root -p > bookDB.sql
      mysql -u root -p --host *rds-endpoint* < bookDB.sql

- Puedes verificar si la migración fue exitosa

      mysql -u root -p --host *rds-endpoint*
      show databases;
      show tables;
      SELECT * FROM Books;

<hr>

8. Por último, puedes probar si la aplicación se conecta a la base de datos en AWS RDS.
