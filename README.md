# Practica 1 (Big Data Architecture)
## Idea general
Recoger datos de eventos de Madrid, en mi caso de la página de www.esmadrid.com/agenda-musica-madrid y ver como estan los precios de la zona para poder hospedarse cerca en algún apartament de Airbnb.

## Nombre del Producto
Recomendador de apartamentos y situacion de precios en la zona cercana al lugar del concierto.

### Estrategia del DAaaS
Voy a presentar un informe mensual en que se presentan los datos de los aparatmentos de Airbnb de Madrid más cercanos al evento. Para realizar esta prctica hemos utilizado herramientas en la nube para el manejo de los datos y alguna herramienta en el equipo local para hacer una depuración del fichero de airbnb.

### Arquitectura

Arquitectura Cloud basada en Scrapy + Google Cloud Storage + HIVE + Dataproc

Se ha realizado Crawler con #Scrapy + Geolocalizacion con #Geopy que recogemos los datos de la página www.esmadrid.com/agenda-musica-madrid, como los datos del lugar 
