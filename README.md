# Descripcion
Documentacion Update Redhat SSO from 7.4 to 7.6 en cluster OpenShift

Los pasos a seguir han sido los siguientes en el proyecto de TEST de Openshift

# STEP01: cliente openshift

Instalar cliente oc en local para poder gestionar los recursos de OpenShift.

- Bajarse el [cliente oc para tu arquitectura](https://access.redhat.com/downloads/content/290). En mi caso el fichero escogido es: **OpenShift v4.11 Linux Client** e instalarlo localmente.

- Login al cluster Openshift con tus credenciales

```
oc login
```

- Configura Proyecto por defecto:

```
oc project arquitecturades
```

# STEP02: borrar todos los recursos actualmente existentes relacionados con RH-SSO bajo el proyecto Arquitectura Desarrollo:

Para acceder al Web Admin del cluster de Openshift seleccionaremos la opción Active_Directory_Consum y utilizaremos las credenciales nuestras bajo la organización de consum.

Accedemos al proyecto Arquitectura Desarrollo de OpenShift llamado *arquitecturades* y borramos todos los recursos ya existentes version 7.4 del RH-SSO. Estos recursos son:

- Aplicación asociada al RH-SSO.
- Aplicación asociada a la base de datos PostgreSQL del RH-SSO.
- Volumen de datos asociada a la instancia RH-SSO en donde desplegar los modulos custom del RH-SSO, entre otros el mapper de AGUA.
- Volumen de datos asociada a la base de datos PostgreSQL del RH-SSO.

Los dos primeros recursos podrán ser borrados directamente desde la vista de aplicaciones del Web Admin del RH-SSO. Los volumenes deberán de ser buscados desde la vista de busqueda de recursos bajo el tipo *persistence volumne claims* (PVC) de OpenShift con el texto *sso* en su interior, veremos que hay dos volumenes uno asociado a PostgreSQL y el otro a la carpeta de despliegues del RH-SSO.

# STEP03: Importar ultimo deployment template para el RH-SSO

Si buscamos la ultima version del RH-SSO veremos que esta es la 7.6.4-GA y que ya existe un deployment template para la misma, de todas formas se baja local este template desde el repo ofical de Redhat en formato json y se importa bajo el repositorio de deployments templates de OpenShift. Tenemos que tener en cuenta que mno existe template que solo instancie el RH-SSO y podamos configurar la DB externamente. Por ello instanciaremos la version RH-SSO con PostgreSQL por defecto y despues modificaremos el Deployment Config creado por el template para conectarlo externamente a la DB correspondiente:

![Openshift Desployment Template Repository](./images/template_repository.png "Openshift Desployment Template Repository")

# STEP04: Desplegar el deployment template para el RH-SSO

Ahora vamos a desplegar la nueva instancia para el RH-SSO escogiendo el template recien registrado llamado: Red Hat Single Sign-On 7.6 on OpenJDK + PostgreSQL (Persistent) for OpenShift 4.X. Lo único que debemos tener en cuenta a la hora de desplegar este template en el proyecto de Arquitectura DES es que configuremos:

- **DB Username**: configurar el nombre de usuario de la DB ya desplegada externamente.
- **DB Password**: configurar la password de usuario de la DB ya desplegada externamente.
- **DB Database Name**: configurar el nombre de la base de datos ya desplegada externamente.

![Openshift Desployment Template Parameters](./images/template_parameters.png "Openshift Desployment Template Parameters")

Tras unos minutos las dos aplicaciones estarán desplegas bajo el nombre sso y sso-postgresql

![Openshift Desployment Template Applications](./images/template_aplications.png "Openshift Desployment Template Applications")

**Nota**: la URL de la DB no se puede configurar gráficamente, pues RH-SSO utiza jdni para definir el Datasource(username, password, db y url). Debemos de desplegar el RH-SSO localmente con la DB y despues modificar el Deployment Config creado y añadir una variable de entorno que sobrescriba la configuración del jdni posteriormente.

# STEP05: Crear un nuevo volumen para los mappers

Debemos de crear un nuevo persistence volume claim para crear un volumen atachado a nuestra instancia de RH-APP en donde guardar los custom mappers. Para ellos accedemos a la opción de Search del Web Admin y buscamos recursos de tipo Persistence Volume Claim (PVC) y creamos este recurso con estas características:

![Openshift Persistence Volume Claim](./images/persistence_volumen_claim.png "Openshift Persistence Volume Claim")

Una vez creado se pueden ver el nuevo volume creado bajo la lista del proyecto

![Openshift Persistence Volume Claim List](./images/persistence_volumen_claim_list.png "Openshift Persistence Volume Claim List")

# STEP06: Reconfigurar la aplicación del SSO

Ahora que tenemos la aplicación del SSO instanciada y el PVC creado vamos a configurar dos cosas en el DeploymentConfig creado por el Template al ser instanciado. 
- Añadir la variable de entorno llamada **SSO_POSTGRESQL_SERVICE_HOST**, esta variable de entorno permite configurar la IP de la base de datos externa donde está corriendo la DB PostgreSQL de RH-SSH con el schema de la antigua versión del RH-SSO 7.4.

- Atachar un volumen llamado **fs-sso-mapper** antes creado bajo el path **/opt/eap/standalone/deployments** de tipo lectura/escritura que es donde RH-SSO recoge los nuevos custom mappers.
  
Aqui podemos ver la nueva variable de entorno definida en el DeploymentConfig desde el Admin Web:

![DeploymentConfig DB Host](./images/deploymentConfig_DB_Host.png "DeploymentConfig DB Host")

Aqui se puede ver como se configura el nuevo volume claim

![DeploymentConfig Volume 01](./images/deploymentConfig_Volume_01.png "DeploymentConfig Volume 01")

Y aqui como se utilizar ese PVC para ser para ser atachado a la instancia de RH-SSO en el path correspondiente
![DeploymentConfig Volume 02](./images/deploymentConfig_Volume_02.png "DeploymentConfig Volume 02")

Una vez modificado el DeploymentConfig se guarda y tras unos segundos la instancia hace un redespliegue y pòr un lado se conecta ahora a la nueva DB que comenzará a actualizar el schema de la DB de la 7.4 a la 7.6 correctamente y al mismo tiempo se crea el nuevo volumen generado por el PVC en el path correspondinte como se puede ver en los nuevos recursos del POD regenerado:

![POD Volumenes](./images/POD_volumenes.png "POD Volumenes")

# STEP07: Desplegar los custom mappers en el volumen

Ahora ya con la nueva instancia desplegada del RH-SSO conectada al nueva DB de PostgreSQL y con el volumen desplegado y atachado correctamente a la instancia. Solo nos queda desplegar los custom mappers que debemos desplegar en la nueva instancia son:

- agua-protocol-mapper-v1.1.0.jar: mapper para conectarse al mapper de agua encargado de recuperar los roles de un id de usuario e injectarlos en el JWT emitido por el RH-SSO
- impersonate-protocol-mapper.jar: mapper para gestionar la impersonación de usuarios

Para ello debemos de recuperar estos recursos  jars de la instancia de RH-SSO desplegada igualmente en el proyecto de TEST del cluster a local y despues subir las misma al entorno de DES bajo la nueva instancia. Usaremos el cli de oc de OpenShift para ello:

Para recuperar los recursos de la instancia de TEST debemos primero colocarnos en este proyecto y bajarnos los recursos a local como sigue, utilizando como origen el nombre del POD de la aplicaión del RH-SSO del proyecto TEST

```
oc proyecto arquitecturatest
oc rsync sso-11-xnw5b:/opt/eap/standalone/deployments ./
```

Despues d ebajarnos los recursos a local, debemos de borrar los ficheros temporales con extensión  deployed, pues son ficheros temporales creados por el servidor de aplicacioens del RH-SSO que serán creados despues tras el despliegue y por ello no deben de subirse a la nueva instancia del RH-SSO de Desarrollo.

Ahora subimo estos recursos (los dos jars antes descritos) de local al volumen de la nueva instancia RH-SSO, debemos de utillizar el nombre del POD y poner la capeta de destino correspondiente en donde queremos guardar estos recursos como sigue:

```
oc rsync . sso-3-mcdhb:/opt/eap/standalone/deployments
```

Tras unos segundos y si nos conectamos por el terminal al POD y vamos a est carpeta de despliegue veremos que aparcen dos ficheros temporales que indican que se han desplegado los jars correctamente y ya estan disponibles para ser utilizados.

# STEP08: Chequeo de la nueva instancia RH-SSO desde el Web Admin:

Ahora que ya esta la nueva instancia actualizada a la version 7.6 con la base de datos PostgreSQL desplegada externamente con el schema ya actualizado igualmente y con los mappers custom desplegados vamos a conectarnos a la Web Admin del RH-SSO y ver que todo esta correcto. Para conectarnos podemos coger las credenciales de Administrador que pueden ser consultadas en las variables de entorno generadas con esta informacion desde la aplicación de sso.

Una prueva que podemos hacer es visualizar el cliente WEB_APP por ejemplo y ver si dentro de los mappers aparecen los que hemos desplegado previamente:

![Cliente RH-SSO con sus mappers](./images/Cliente_RH_SSO_Mappers.png "Cliente RH-SSO con sus mappers")

Se peude ver como esta configurado y listo para ser utilizdo por el SSO cuando intetemos emitir un token.

# STEP09: Pruebas end-to-end con la nueva instancia:

Finalmente vamos a realizar una prueba end-to-end para ver que el sistema emite Tokens y este inyecto los roles del usuario procedentes de AGUA utilizando el custom mapper desplegado en el RH-SSO. Ejecutamos este comando:

Aqui se puede ver el comando curl para obtener un token

```
curl --location 'https://sso-arquitecturades.opsdev.consum.es/auth/realms/test/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--header 'Cookie: 33420d2c1a70acbb5a98bc5413af6ebf=cddedd34e544ff4275462f40c9903207' \
--data-urlencode 'client_id=WEB_APP' \
--data-urlencode 'client_secret=5fd91a85762f4a7fb5ee3fc8fb648e1a' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'username=<USER_NAME>' \
--data-urlencode 'password=<USER_PASSWORD>

```

Y el resultado del mismo

![Test end-to-end](./images/end_to_end.png "Test end-to-end")

Si inspeccionamos el JWT obtenido podremos ver como correcyamente bajo el claim de authorities del Token exiten los roles que han sido definidos en AGUA, por lo que podemos concluir que la nueva instancia de RH-SSO version 7.6.4 GA con los custom mappers funciona correctamente

![AGUA user roles](./images/agua_user_roles.png "AGUA user roles")