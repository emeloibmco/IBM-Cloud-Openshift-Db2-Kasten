# Documentación de Configuración para DB2 en OpenShift

## Pre-Requisitos :pencil:
- Contar con un clúster de Openshift en [IBM Cloud](https://cloud.ibm.com/kubernetes/catalog/create?platformType=openshift&catalog_query=aHR0cHM6Ly9jbG91ZC5pYm0uY29tL2NhdGFsb2c%2FY2F0ZWdvcnk9Y29udGFpbmVycw%3D%3D)
- Contar con una instancia de [IBM Cloud Object Storage](https://cloud.ibm.com/objectstorage/create)

## Instalacion DB2

### Antes de empezar

Revisar que exista el storageClass referenciado en la PVC.
Añade al storageClass usado la sgt configuracion:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ibmc-vpc-block-10iops-tier
mountOptions:
  - suid
```

### Crear namespace
Crea un nuevo proyecto (namespace) para la implementación de DB2.
```bash
oc new-project db2-test-5
```
### Crear scc
Aplica la configuración de SCC desde el archivo scc.yml.
```bash
oc apply -f scc.yml
```
### Crear service account
Crea una cuenta de servicio necesaria para el acceso a los recursos.
```bash
oc create serviceaccount db2-sa -n db2-test-5
```
### Realizar binding entre scc y service account
```bash
oc create rolebinding db2-sa-scc-binding \
--clusterrole=system:openshift:scc:db2u-scc \
--serviceaccount=db2-test-5:db2-sa \
--namespace=db2-test-5
```
### Troubleshooting - securityContext.privileged: true
Para evitar el error es necesario ejecutar el siguiente comando.
```bash
oc adm policy add-scc-to-user privileged -z db2-sa -n db2-test-5
```
###  Crear pvc
Aplica la configuración de PVC desde el archivo pvc.yaml.
```bash
oc apply -f pvc.yaml
```
### Crear deployment 
Crea el deployment de DB2 usando el archivo deployment.yaml.
```bash
oc apply -f deployment.yaml
```
### Verifica service account en los pods
Revisa la configuración de la cuenta de servicio en los pods existentes.
```bash
oc describe pod <pod-name> -n db2-test-5
```
### Login
Entrar al pod del db2 por la consola de openshift
```shell
su - db2inst1

db2
```
```db2
CONNECT TO testdb

CREATE TABLE employees(id INT PRIMARY KEY NOT NULL, name VARCHAR(50), salary DECIMAL(10, 2))

INSERT INTO employees (id, name, salary) VALUES (1, 'Alice', 50000.00)

SELECT * FROM employees

```

## Export dabase a COS

### Crear instancia de Cloud Object Storage

### Crear secret apartir de la instancia de COS

```shell

oc create secret generic cos-write-access --type=ibm/ibmc-s3fs --from-literal=access-key=<access_key_ID> --from-literal=secret-key=<secret_access_key>    

```

### Instalar plugin cos 

Seguir el siguiente [instructivo](https://cloud.ibm.com/docs/openshift?topic=openshift-storage_cos_install)

### Crear pvc para el cos

En el cli aplicar el archivo pvc-cos.yaml, realizando los cambios pertinentes al secret y el namespace donde se encuentra en yaml.

```shell
oc apply -f pvc-cos.yaml   
```
### Crear o actualizar el deployment

Aplicar el archivo deployment4backup.yaml

```shell
oc apply -f deployment4backup.yaml
```
### Dar permisos al usuario db2inst1

Una vez actualizado el pod, actualizar los permisos para que el usuario db2inst1 pueda realizar la copia del db2 sobre el volumen de cos.

```shell

id db2inst1 #obtener group id

chown -R <group id> /backup
chmod -R 770 /backup
```

### Exportar db2
Entrar al cli del pod

```shell

su -db2inst1

cd /backup #backup el directorio perteneciente al cos especificado en el yaml del deployment

db2move <db name> EXPORT

```

## Backup con Kasten

### Ajuste pre-instalacion
Para realizar backup's de aplicativos que tengan persistent volume es necesario anadir esta anotacion al YAML del VolumeSnapshotClass que se va a utilizar.

![image](https://github.com/user-attachments/assets/14da9cd0-f903-41b4-912b-87de38982cba)


### Instalar Kasten 
Sigue el instructivo hasta la configuración del location profile. [Manual de instalación Kasten](https://github.com/emeloibmco/Red-Hat-Open-Shift-Kasten-Backup)

### Creación y ejecución de una política de Backup
Se debe crear una policy con la siguiente configuracion, para tomar configuraciones necesarias para el DB2 que pueden no estar en el namespace.

![image](https://github.com/user-attachments/assets/5f923a1a-5e1e-429b-b138-1cfdb78e3350)

### Configuracion de política de restauración de Backup

Sigue el instructivo para la configuración de la política de import. [Manual de instalación Kasten](https://github.com/emeloibmco/Red-Hat-Open-Shift-Kasten-Backup)

## Referencias

### Documentacion img

https://hub.docker.com/r/ibmcom/db2

### Documentacion Adicional(scc)

https://www.ibm.com/docs/en/db2/11.5?topic=db2-deploying

### Documentacion plugin

https://cloud.ibm.com/docs/openshift?topic=openshift-storage_cos_install
