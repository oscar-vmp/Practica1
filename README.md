# Practica 1 (Big Data Architecture)
## Idea general
Recoger datos de eventos de Madrid, en mi caso de la página de www.esmadrid.com/agenda-musica-madrid y ver como estan los precios de la zona para poder hospedarse cerca en algún apartament de Airbnb.

## Nombre del Producto
Recomendador de apartamentos y situacion de precios en la zona cercana al lugar del concierto.

### Estrategia del DAaaS
Voy a presentar un informe mensual en que se presentan los datos de los aparatmentos de Airbnb de Madrid más cercanos al evento. Para realizar esta prctica hemos utilizado herramientas en la nube para el manejo de los datos y alguna herramienta en el equipo local para hacer una depuración del fichero de airbnb.

### Arquitectura

Arquitectura Cloud basada en Scrapy + Google Cloud Storage + HIVE + Dataproc

Se realizada un Crawler con Scrapyde la página www.esmadrid.com/agenda-musica-madrid, como en el Scrapy solo disponemoS de la direccion del evento, mediante el Api de Geopy en el mismo proceso obtenemos la Geolocalizacion del lugar del concierto.

Antes de insertar el fichero de Airbnb en Google Storage, primero he realizado un ETL en el dataset para poder cargarle luego con ciertas garantías, ya que el fichero en cuestion tenía muchos saltos de líneas dentro de los campos que impide cargar el fichero en HIVE. Esto proceso lo he realizado con la herramienta Qlikview, que me ha permitido limpiar los datos dejandolos listos para cargar en Google Storage.

Una vez tenemos los dos ficheros el de Scrapy y el de Airbnb depurado, he realizado un proceso de crear eun fichero de distancias. Lo que he hecho es leer de Google Storage los dos ficheros y generar una rutina en el que cada evento obtenemos su Geolocalización, despues lo cruzamos con cada unna de las Geolocalizaciones del fichero de Airbnb y calculamos la distancia con cada uno de los aparatmentos en este cso calculado en metros. Para realizar este cálculo he vuelto a utilizar el Api de Geopy. Una vez generado estos datos los hemso guardado en un fichero csv. Para realizar esta tarea tambien la hemos ejecutado mediante una Cloud Function. 

Una vez tenemos los tres ficheros en nuestro segmento de Google Storage, vamos a crear tres tablas en HIVE. Aqui por ejemplo, he realizado una query que para un conciento en concreto me saque todos los apartamentos que se encuentren a una distancia del evento de un radio de 1 Km.

El resultado de la busqueda SQL la almacenaremos en un fichero que se colocará en nuestro segmento de Google Storage.
