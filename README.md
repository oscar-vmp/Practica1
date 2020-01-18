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

Para realizar que funcione esta Arquitectura vamos a de pender de un operador, que primero desde su móvil active el proceso de Crawler que está creada como una Cloud Funtion, después de finalizar el proceso y el fichero se ha creado en la carpeta de input/scrapy de mi segmento en el Cloud Storage, el operador ejecutará la Cloud Funcion que crea la tabla de distancias y la guarada en la carpeta input del Clod Storage. Este proceso se puede hacer con una periodicidad mensual, puesto que los conciertos no varían mucho porque suelen estar programados con bastante antelación.
Una vez estan cargados los datos, el operador levantará un Cluster y creará las tareas siguientes:
- Crear la tabla de __airbnb__
- Crear la tabla de __eventos__
- Crear la tabla de __distancias__
- LOAD DATA INPATH 'gs://segmento_practicas_ov/input/airbnb/airbnb_depurado.csv' INTO TABLE airbnb;
- LOAD DATA INPATH 'gs://segmento_practicas_ov/input/scrapy/conciertos.csv' INTO TABLE eventos;
- LOAD DATA INPATH 'gs://segmento_practicas_ov/input/distancias.csv' INTO TABLE evento_piso ; 
- Crear el SQL y caragar el resultado en el segmento de Cloud Storage

Cuando se finalice el proceso el operador apagará el Cluster.
Una vez se tenga el fichero de salida se procedera a ttrabajar con los datos y presentarlos.

![Grafico del proceso](/graficos/grafico.jpg "Grafico del proceso")






	from google.cloud import storage
	from scrapy.crawler import CrawlerProcess
	from datetime import datetime
	from geopy.geocoders import Nominatim
	from geopy.exc import GeocoderTimedOut

	import scrapy
	import json
	import tempfile
	TEMPORARY_FILE = tempfile.NamedTemporaryFile(delete=False, mode='w+t')

	def upload_file_to_bucket(bucket_name, blob_file, destination_file_name):
    	"""Uploads a file to the bucket."""
   	 storage_client = storage.Client()
   	 bucket = storage_client.get_bucket(bucket_name)
    	blob = bucket.blob(destination_file_name)
   	 blob.upload_from_filename(blob_file.name, content_type='text/csv')



	class ConciertosMadridSpider(scrapy.Spider):
		name = 'conciertosmadrid'
    		start_urls = ['https://www.esmadrid.com/conciertos-no-te-puedes-perder-madrid-2020/']
    
		id=1
		COUNT_MAX = 80
		count = 0	
	

		def parse(self, response):
        	
			title_text = response.css('h1.field.field-name-title-field.field-type-text.field-label-hidden div.field-items div.field-item.odd.first.last::text').extract_first()
			dia_text = response.css('div.field.field-name-field-when.field-type-text.field-label-above::text').extract_first()
			tipo_text=response.css('div.fieldset-wrapper div.group-wrapper-direction.field-group-div div.field.field-name-field-via-type.field-type-taxonomy-term-reference.field-label-hidden div.field-items div.field-item.odd::text').extract_first() 
			dire_text= response.css('div.fieldset-wrapper div.group-wrapper-direction.field-group-div div.field.field-name-field-address.field-type-text.field-label-hidden div.field-items div.field-item.odd::text').extract_first()
			cp_text= response.css('div.fieldset-wrapper div.group-wrapper-direction.field-group-div div.field.field-name-field-postal-code.field-type-text.field-label-hidden div.field-items div.field-item.odd::text').extract_first()
			zona_text= response.css('div.fieldset-wrapper div.field.field-name-field-turisitic-zone.field-type-taxonomy-term-reference.field-label-above::text').extract_first()
			direccion_text = "%s %s %s" % (tipo_text,dire_text,cp_text)
			
			evento="concierto"
			geolocator = Nominatim(user_agent="MyApp")
			dir_geopy = "%s %s , %s" % (tipo_text,dire_text,"Madrid")
			location= None
			latitud=0.0
			longitud=0.0
			valor=False
			try:
			location = geolocator.geocode(dir_geopy, timeout=3)
		except GeocoderTimedOut: 
			valor=True
   
		if valor:
			try:
				dir_geopy = "%s , %s" % (dire_text,"Madrid")
				location = geolocator.geocode(dir_geopy, timeout=3)
			except GeocoderTimedOut: 
				latitud=0.0
				longitud=0.0            
		if location:
			latitud=  location.latitude
			longitud=  location.longitude
		else:
			try:
				dir_geopy = "%s , %s" % (dire_text,"Madrid")
				location = geolocator.geocode(dir_geopy)
				if location:
					latitud=  location.latitude
					longitud=  location.longitude
				else:
					latitud=0.0
					longitud=0.0    
			except GeocoderTimedOut: 
				latitud=0.0
				longitud=0.0   
        
		if dia_text is not None:
			TEMPORARY_FILE.writelines(f"{self.id}|{evento}|{title_text}|{dia_text}|{direccion_text}|{cp_text}|{zona_text}|{latitud}|{longitud}\n")
			self.id = self.id + 1        


         # Aqui hacemos crawling (con el follow)
		for next_page in response.css('div.ds-1col.node.node-event.view-mode-grid_2col.clearfix div.field-group-link-format.group_link_wrappergroup-link-wrapper.field-group-link a'):
			self.count = self.count + 1
			if (self.count < self.COUNT_MAX):
				yield response.follow(next_page, self.parse)
                
def activate(request):
	now = datetime.now() 
	request_json = request.get_json()
	BUCKET_NAME = 'segmento_practicas_ov'
	DESTINATION_FILE_NAME = 'input/scrapy/conciertos.csv'
	process = CrawlerProcess({'USER_AGENT': 'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)'})
	process.crawl(ConciertosMadridSpider)
	process.start()    
	TEMPORARY_FILE.seek(0)
	upload_file_to_bucket(BUCKET_NAME, TEMPORARY_FILE, DESTINATION_FILE_NAME)
	TEMPORARY_FILE.close()
	return "Success!"
