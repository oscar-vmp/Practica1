# Practica 1 (Big Data Architecture)
## Idea general
Recoger datos de eventos de Madrid, en este caso de la página de www.esmadrid.com/agenda-musica-madrid y ver como estan los precios de la zona para poder hospedarse cerca en algún apartamento de Airbnb.

## Nombre del Producto
Recomendador de apartamentos y situacion de precios en la zona cercana al lugar del concierto.

### Estrategia del DAaaS
Se va a presentar un informe mensual en que se presentan los datos de los aparatmentos de Airbnb de Madrid más cercanos al evento. Para realizar esta practica se han utilizado herramientas en la nube para el manejo de los datos y alguna herramienta en el equipo local para hacer una depuración del fichero de airbnb.

### Arquitectura

Arquitectura Cloud basada en Scrapy + Google Cloud Storage + HIVE + Dataproc

Se realizada un Crawler con Scrapy de la página www.esmadrid.com/agenda-musica-madrid, como en el Scrapy solo se dispone de la direccion del evento, mediante el Api de **Geopy** en el mismo proceso se obteninela Geolocalizacion del lugar del concierto para posteriores procesos.

Antes de insertar el fichero de Airbnb en Google Storage, primero ha realizado un ETL en el dataset para poder cargarle luego con ciertas garantías, ya que el fichero en cuestion tenía muchos saltos de líneas dentro de los campos que impide cargar el fichero en HIVE. Esto proceso se ha realizado con la herramienta **Qlikview**, que ha permitido limpiar los datos dejandolos listos para posteriormente se puedan cargar en HIVE.

Una vez se tienen los dos ficheros, el de Scrapy y el de Airbnb depurado, se ha realizado un proceso de crear un fichero de distancias. Lo que se ha hecho es leer de Google Storage los dos ficheros y generar una rutina en el que cada evento obtenemos su Geolocalización, después lo cruzamos con cada unna de las Geolocalizaciones del fichero de Airbnb y calculamos la distancia con cada uno de los aparatmentos, en este caso calculado en metros. Para realizar este cálculo se ha vuelto a utilizar el Api de **Geopy**. Una vez generado estos datos se han guardado en un fichero csv. Para realizar esta tarea también se ha ejecutado mediante una Cloud Function. 

Una vez se tienen los tres ficheros en nuestro segmento de Google Storage, se van a crear tres tablas en HIVE. 

![Tablas de HIVE](/graficos/tablas.jpg "Tablas de HIVE")

Aqui por ejemplo, se ha realizado una query que para un concierto en concreto saque todos los apartamentos que se encuentren a una distancia del evento de un radio de 1 Km.

El resultado de la busqueda SQL se almacenará en un fichero que se colocará en nuestro segmento de Google Storage.

### Operating Model


