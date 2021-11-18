# Segmentación: Ingesta de Datos

# Tabla de Contenido

1. [Resumen](#resumen)
2. [Preprocesos](#preprocesos)
3. [Arquitectura](#arquitectura)

## Resumen

[(Volver al Inicio)](#tabla-de-contenido)

**Alicorp**, cuenta con una amplia cartera de clientes, estando entre ellos las __*bodegas*__. Se ha realizado preprocesos que permiten al interesado obtener bases de datos de gran ayuda para posteriores procesos que tienen como resultado final el cluster al que corresponde el cliente, la estrategia adecuada que se recomienda aplicar y un sistema de recomendación.

## Preprocesos

[(Volver al Inicio)](#tabla-de-contenido)

A continuación se procederá a describir siete preprocesos para obtener las bases de datos que ayudaran a procesos posteriores, y cada preproceso contiene su respectivo flujo de procedimientos.

1. **VENTAS:** La data es obtenida de **QlikSense(.qvd)**, y es trabajada todos los miércoles con la información recolectada de una semana anterior,el siguiente paso es almacenar toda esa data en un servidor, el cual transforma la data en formato csv, teniendo un total de siete datas, a continuación se realiza procesos ya automatizados que cargan la base de datos a **Google Cloud Storage**, después se ejecuta un trigger en **Google Cloud Functions** y se finaliza con la ejecución del gestor de base de datos en PostgreSQL. En la siguiente imagen se observa el flujo completo:

![Ventas](img/automated.jpg "Ventas")

2. **MAESTRO PRODUCTO:** La obtención de la base de datos sigue el mismo proceso que la anterior y en la siguiente imagen se observa el flujo completo:

![Ventas](img/automated.jpg "Ventas")

3. **MAESTRO CLIENTE:** Estos datos se encuentran en formato csv y son facilitados por una persona a cargo. Estos datos contiene el código del cliente, el nombre y la boca salida (B2B y B2C). Y el flujo del proceso esta en la siguiente imagen.

![Maestro_Cliente](img/automated.jpg "Maestro_Cliente")

4. **TERRITORIO:** El proceso de la obtención de la base de datos es igual al anterior proceso, y esta contiene el código territorio, el nombre, ID de fuerza de venta y la fuerza de venta.

![Territorio](img/automated.jpg "Territorio")

5. **DISTRIBUIDOR EXCLUSIVO:** La obtención de la base de datos sigue el mismo proceso, donde se encuentra el ID del dex, nombre del dex, ID del cliente actual y el nombre del cliente actual, en la siguiente imagen se observa el flujo completo:

![Distribuidor Exclusivo](img/automated.jpg "Distribuidor Exclusivo")


## Conceptos Básicos de Apache Beam

A continuación se presenta un diagrama de cómo se representa un pipeline en Apache Beam.

![Conceptos Basicos](img/basic_concepts.png)

### PCollection

Un "PCollection" es básicamente la representación de un conjunto de datos. En nuestro caso se usa como una variable "wrapper" que nos permite almacenar datos que pueden ser usados y accedidos para las siguientes etapas de nuestro pipeline.

### ParDo

Es una clase que se implementa en Apache Beam para realizar una transformación **genérica** a los elementos de un PCollection. A continuación se muestra un ejemplo que como se implementaría un simple pipeline usando dos tareas parDos (A y B).

```python
class A(beam.DoFn):
  def process(self, element):
    # La variable element es importante, ya que ese sería el input de entrada de nuestro nodo o tarea A en el Pipeline

    a = hard_task_running1()  # Ejemplo a == 1
    b = hard_task_running2()  # Ejemplo a == 2
    yield a, b  # Estas variables que para el Nodo A son la salida, sería los elementos de entrada para otro nodo, es decir del B
```
```python
class B(beam.DoFn):
  def process(self, element):
    # La salida de A es la entrada de nuestro nodo B
    a, b = element
    print(a)   # Si hacemos print, el valor de element sería 1

    c = hard_task_running3()

    yield c # Se puede pasar esta variable va a ser a su vez el dato de entrada para otro nodo. 
```
Unimos los nodos de esta manera en el archivo principal
```python
p = beam.Pipeline()

(p
  | 'Tarea A' >> beam.ParDo(A())
  | 'Tarea B' >> beam.ParDo(B())   
)

p.run()
```

## Arquitectura

[(Volver al Inicio)](#tabla-de-contenido)

A continuación se muestra la arquitectura empleada. Como se puede ver, el pipeline que se define en python usando el SDK de Apache Beam permite finalmente definir una plantilla en formato .json. Esto último es lo que finalmente Dataflow usaría para correr el pipeline, ya que en este mismo se definen todos los pasos que se deben de ejecutar con la información dada.

El pipeline de Dataflow puede ser "triggeado" o lanzando mediante distintas formas como se puede ver en el diagrama. Esto es gracias al API Endpoint que expone la infraestructura de Dataflow. Para conocer más acerca de esto, se puede ver [la documentacion del API de Google Dataflow](https://cloud.google.com/dataflow/docs/reference/rest).

![Arquitectura](img/architecture.png "Arquitectura")

## Pipeline

[(Volver al Inicio)](#tabla-de-contenido)

Se presenta el diagrama del flujo de tareas construido para este pipeline. Uno de los motivos de diseñar de esta manera el flujo completo (puede terminar siendo gigante), se debe a la **limitación** de usar templates en Dataflow, ya que una vez creado y subido al gcs, no se puede modificar el flujo. Si se quiere cambiar el flujo, se deberá de modificar el código python y otra vez actualizar el template.

![Pipeline](img/pipeline.png)

### Tareas Principales

- **Begin pipeline with initiator:** inicializa el pipeline.
- **Load enviroment variables:** se cargan las variables de entorno del secret manager.
- **Get DB Config:** se obtiene los parámetros de conexión para la base de datos.
- **Get file info:** se obtiene los datos del archivo CSV.
- **Extract:** se lee el archivo CSV del bucket.
- **Log update:** se actualiza el registro de log que ha iniciado correctamente el pipeline.
- **Filter:** se verifica si el archivo pertenece al folder (customers-commercial, customers-dex, etc). Si es así, se procede con el flujo, caso contrario, no sigue el flujo.
- **Format string to object:** convierte la linea de texto en un objeto json en memoria.
- **Transform:** realiza las transformaciones necesarias a los elementos. Ejemplo, operaciones como: upper, trim, etc.
- **Format object to string:** convierte el objecto json en un string.
- **Load:** realiza el proceso de carga de datos, tanto a la base de datos Posgrest como Big Query.
- **Flatten:** une todas las bocas de salida en un solo punto.
- **Log Update Finish:** actualiza en el db log que se ha terminado correctamente el flujo. 

## Comandos Makefile:
- **make gdf:** genera el template .json del pipeline y lo almacena en un bucket.
- **make gdf-run:** lanza el pipeline en dataflow leyendo el template. (previamente debe de haberse generado el template y subido a un gcs).
- **make local:** corre el pipeline de manera local (**NO** usa dataflow, ni templates).
- **create-bq-dataset:** crea un dataset en BQ.
- **delete-bq-dataset:** elimina el dataset creado en BQ.
- **create-gcs-scheduler-start-sunday:** crea un scheduler para prender la VM el día domingo.
- **create-gcs-scheduler-stop-sunday:** crea un scheduler para apagar la VM el día domingo.
- **create-gcs-scheduler-start-thursday:** crea un scheduler para prender la VM el día jueves.
- **create-gcs-scheduler-stop-thursday:** crea un scheduler para apagar la VM el día jueves.