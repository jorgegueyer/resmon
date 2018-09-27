# Servicios Rest, Mongo, y ConfigMaps de Kubernetes

## Objetivos

El objetivo de este tutorial es configurar una aplicación Spring Boot a través de ConfigMaps de Kubernetes. Se pretende mostrar como se puede externalizar la configuración de una aplicación Spring Boot a traves del uso de los ConfigMaps de Kubernetes. Para ello, se va a externalizar la URI de acceso a una base de datos MongoDB mediante el uso de ConfigMaps.

Existen dos formas de externalizar la configuración de una aplicación Spring Boot a través de ConfigMaps:

1. [Autocarga de ConfigMap](#autocarga-de-configmap)
2. [Carga de ConfigMap mediante volumenes](#carga-de-configmap-mediante-volumen)

Antes de poder mostrar como se hace uso de los ConfigMaps, se deben realizar una serie de instalaciones y acciones previas:

## Pre-requisitos

Tener [Docker](https://docs.docker.com/install/), [MongoDB](https://docs.mongodb.com/manual/installation/) y [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) instalado.

Además es necesario crear una base de datos con el nombre de **"sample"**.

### Uso de Minikube

Iniciar el servicio de minikube:
 
`minikube start`

**Importante!** Hay que vincular minikube y el servicio de Docker:

`eval $(minikube docker-env)`

Para poder leer los ConfigMaps desde un Pod, hay que crear un rolebinding para asignarle permisos de edición:

`kubectl create rolebinding serviceaccounts-edit --clusterrole=edit --group=system:serviceaccounts:default --namespace=default`

### Preparar proyecto para su despliegue y ejecución

En primer lugar, hay que cambiar la ubicación al directorio raiz del proyecto:

`cd /path/to/resmon`

Ejecutar el comando clean install de Maven para realizar la compilación y despliegue de la aplicación y crear el contenedor. La generación de este último es posible gracias a que este proyecto dispone del archivo Dockerfile donde está todos los datos de configuración del contenedor:

`./mvnw clean install -DskipTests`

Comprobar que existe la imagen del contenedor que se acaba de crear:

`docker images | grep resmon`

Una vez llegados a este punto, se puede optar por los dos métodos para la externalización de la configuración mediante ConfigMaps:

### Autocarga de ConfigMap

Una vez se ha creado el contenedor, se debe crear un ConfigMap en Kubernetes que tenga como nombre el mismo nombre que se le ha dado al contenedor. Esto debe ser así ya que cuando Kubernetes despliega y ejecuta el contendor, busca si alguno de los identificadores de los ConfigMaps conincide con el nombre del contenedor.

Se crea el ConfigMap:

`kubectl create configmap resmon --from-file=/path/to/resmon/configmap/application.properties`

Como se puede observar, el identificador del ConfigMap coincide con el nombre del contenedor, resmon en ambos casos.

Una vez está creado el ConfigMap, se debe crear el deployment de Kubernetes:

`kubectl run resmon --image=resmon:v1 --port=8080`

Se recomienda comprobar si el Pod se está ejecutando correctamente:

`kubectl get pods`

Una vez inicializado el deployment, es necesario exponerlo como un servicio de tipo NodePort:

`kubectl expose deployment resmon --type=NodePort`

Una vez se ha finalizado el proceso de configuración, despliegue, y configuración del contenedor, se comprobará su funcionamiento y el de la aplicación mediante la realización de la siguientes [pruebas](#pruebas)

### Carga de ConfigMap mediante volumen

Se crear el ConfigMap:

`kubectl create configmap resmon-config --from-file=/path/to/resmon/configmap/application.yml`

Ahora el identificador del ConfigMap no coincide con el nombre del contenedor, tiene un nombre personalizado. Para este caso, el ConfigMap se configura a través del deployment del contenedor, indicando qué ConfigMap se quiere usar y en qué directorio dentro del contenedor se quiere desplegar.

En primer lugar se puede ver como en la configuración del contenedor se indica que se desea montar el volumen config-app en el path /config. Justo por debajo, se crean un volumen con nombre config-app que se vincula con el  ConfigMap resmon-config.

```yaml
...
spec:
      containers:
        - name: resmon
          image: resmon:v1          
          ports:
          - containerPort: 8080
            protocol: TCP  
          volumeMounts:
          - name: config-app
            mountPath: /config
      volumes:
        - name: config-app     
          configMap:
            name: resmon-config 
            items:
            - key: application.yml
              path: application.yml              
```

Se puede ver el contenido completo del deployment [aquí]()

Por último, se debe lanzar la creación del deployment y del service:

`kubectl create -f path/to/resmon/deploy/deployment.yaml`

Una vez se ha finalizado el proceso de configuración, despliegue, y configuración del contenedor, se comprobará su funcionamiento y el de la aplicación mediante la realización de la siguientes [pruebas](#pruebas)

### Pruebas

Para comprobar que todos los recursos asociados al contenedor estan funcionando correctamente, se pueden usar los siguientes comandos:

Comprobar que el Deployment esta funcionando correctamente:

`kubectl get deployments`

Comprobar el estado del Pod:

`kubectl get pods` ó `kubectl get po`

Por último, comprobar que existe el Service y qué puerto se le ha asignado:

`kubectl get services` ó `kubectl get svc`

En caso de que se quieran consultar los logs de la aplicación:

`kubectl logs -f [POD_NAME]`

El POD_NAME se puede obtener haciendo una consulta del estado del Pod.

Una vez terminadas las comprobaciones sobre el contenedor, ahora hay que comprobar que la aplicación esta funcionando correctamente:

`minikube service resmon`

Con este comando, minikube abre el navegador por defecto donde se carga como dirección la ip y puerto de la aplicación. Con esta URL, se pueden formar las diferentes URL's que invocan a los direfentes endpoints expuestos por la aplicación.

Inicialmente la base de datos debería estar vacía.
Para crear una nueva persona:

`http://EXTERNAL_IP:PORT/people`

Se trata de una petición POST que debe llevar información en el cuerpo de la petición con el siguiente formato:

```json
  {
    "firstName" : "Pepito",
    "lastName" : "Perez"
  }
```

Para consultar personas, hay diferentes opciones:

1. Consultar todos los registros de personas:
`http://EXTERNAL_IP:PORT/people`

2. Consultar una persona por id:
`http://EXTERNAL_IP:PORT/people/{id}`

3. Consultar personas por el apellido:
`http://EXTERNAL_IP:PORT/people/search/findByLastName?name={lastName}`
