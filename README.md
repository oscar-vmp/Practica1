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


### Desarrollo

El primer proceso a realizar fue depurar el fichero de **airbnb-listings.csv**, ya que como se puede ver en la imagen :

![Fichero airbnb-listings](/imagenes/airbnb-listings.jpg "Fichero airbnb-listings")

Para realizar esta depuracion se ha utilizado la herramienta de **Qlikview**, se han quitado caracteres como retornos de carro, caracteres raros, etc.. El resultado final es el que se presenta a continuación:

![Fichero airbnb-depurado](/imagenes/airbnb_depuerado.jpg "Fichero airbnb-depurado")

El Script realizado para la depuración fue el siguiente:

	LOAD 
		ID, 
		[Listing Url], 
		[Scrape ID], 
		[Last Scraped], 
		KeepChar(Name,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Name,
		KeepChar(Summary,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Summary,
		KeepChar(Space,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Space,
		KeepChar(Description,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Description,
		KeepChar([Experiences Offered],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Experiences Offered],
		KeepChar([Neighborhood Overview],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]() ') as [Neighborhood Overview],
		KeepChar(Notes,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Notes,
		KeepChar(Transit,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Transit,
		KeepChar(Access,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Access,
		KeepChar(Interaction,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Interaction,
		KeepChar([House Rules],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [House Rules],
		[Thumbnail Url], 
		[Medium Url], 
		[Picture Url], 
		[XL Picture Url], 
		[Host ID], 
		[Host URL], 
		KeepChar([Host Name],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Host Name],
		KeepChar([Host Since],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Host Since],
		KeepChar([Host Location],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Host Location],
		KeepChar([Host About],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Host About],
		[Host Response Time],
		[Host Response Rate], 
		[Host Acceptance Rate], 
		[Host Thumbnail Url], 
		[Host Picture Url], 
		[Host Neighbourhood], 
		[Host Listings Count], 
		[Host Total Listings Count], 
		KeepChar([Host Verifications],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Host Verifications], 
		KeepChar(Street,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Street,
		KeepChar(Neighbourhood,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Neighbourhood, 
		KeepChar([Neighbourhood Cleansed],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Neighbourhood Cleansed], 
		KeepChar([Neighbourhood Group Cleansed],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Neighbourhood Group Cleansed], 
		KeepChar(City,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as City, 
		KeepChar(State,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as State, 
		KeepChar(Zipcode,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Zipcode, 
		KeepChar(Market,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Market, 
		KeepChar([Smart Location],'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as [Smart Location], 
		[Country Code], 
		Country, 
		Latitude, 
		Longitude, 
		[Property Type], 
		[Room Type], 
		Accommodates, 
		Bathrooms, 
		Bedrooms, 
		Beds, 
		[Bed Type], 
		Amenities, 
		[Square Feet], 
		Price, 
		[Weekly Price], 
		[Monthly Price], 
		[Security Deposit], 
		[Cleaning Fee], 
		[Guests Included], 
		[Extra People], 
		[Minimum Nights], 
		[Maximum Nights], 
		[Calendar Updated], 
		[Has Availability], 
		[Availability 30], 
		[Availability 60], 
		[Availability 90], 
		[Availability 365], 
		[Calendar last Scraped], 
		[Number of Reviews], 
		[First Review], 
		[Last Review], 
		[Review Scores Rating], 
		[Review Scores Accuracy], 
		[Review Scores Cleanliness], 
		[Review Scores Checkin], 
		[Review Scores Communication], 
		[Review Scores Location], 
		[Review Scores Value], 
		License, 
		[Jurisdiction Names], 
		[Cancellation Policy], 
		[Calculated host listings count], 
		[Reviews per Month], 
		Geolocation, 
		KeepChar(Features,'abcdefghijklmnñopqrstuvwxyzABCDEFGHIJKLMNÑOPQRSTUVWXYZ0123456789.,:-+*/"·!?¿&%$={}[]()áÁéÉíÍóÓúÜ ') as Features
	FROM
	[D:\PRACTICAS\airbnb-listings.csv]
	(txt, utf8, embedded labels, delimiter is ';', msq);


A continuación se detalla las **Cloud Functions** realizadas y ejecutadas:

![Cloud Functions](/imagenes/cloud_functions.jpg "Cloud Functions")

Primera Cloud Function ejecutada es la llamada **function-scrapy-conciertos-1**, que es la encargada de hacer el Crawler y la geolocalizacion: 

El fichro de __requerimients.txt__

	# Function dependencies, for example:
	# package>=version
	google-cloud-storage
	scrapy
	geopy

El fichero __main.py__

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


El resultado se guarda en una carpeta de nuestro segmento de Google Storage: 

![Fichero eventos](/imagenes/conciertos_sc.jpg "Fichero eventos")

Después se ejecuta la siguiente Cloud Function llamada **function-ficheros-1**, con esta Cloud Funtion creamos un fichero de distancias entre el evento y los apartamentsos de Airbnb:

El fichro de __requerimients.txt__

	# Function dependencies, for example:
	# package>=version
	google-cloud-storage
	geopy

El fichero __main.py__
	from google.cloud import storage
	from datetime import datetime

	import json
	import tempfile

	TEMPORARY_FILE = tempfile.NamedTemporaryFile(delete=False, mode='w+t')

	def upload_file_to_bucket(bucket_name, blob_file, destination_file_name):
	    """Uploads a file to the bucket."""
	    storage_client = storage.Client()
	    bucket = storage_client.get_bucket(bucket_name)
	    blob = bucket.blob(destination_file_name)
	    blob.upload_from_filename(blob_file.name, content_type='text/csv')

	def leer_fichero(bucked_name,entrada):
		datos=[]
		storage_client = storage.Client()
		bucket = storage_client.get_bucket(bucked_name)
		blob = bucket.get_blob(entrada)
		downloaded_blob = blob.download_as_string()
		s1 = str(downloaded_blob, encoding = 'utf-8')
		lineas=s1.split('\n')
		for linea in lineas:
			linea = linea.strip()
			campos = linea.split("|")
		
			if len(campos)==9:
				dato = {'id':campos[0],
						'longitud':campos[8],
						'latitud':campos[7]}
				datos.append(dato)
		return datos
   

	def leer_fichero_airbnb(bucked_name,entrada):
		datos=[]
		storage_client = storage.Client()
		bucket = storage_client.get_bucket(bucked_name)
		blob = bucket.get_blob(entrada)
		downloaded_blob = blob.download_as_string()
		s1 = str(downloaded_blob, encoding = 'utf-8')
		lineas=s1.split('\n')
		for linea in lineas:
			linea = linea.strip()
			campos = linea.split(";")
			if len(campos)==88:
				dato = {'id':campos[0],
						'longitud':campos[46],
						'latitud':campos[45]}
				datos.append(dato)
		return datos
	def formatear_linea(dato):
	    linea = []  #una lista con los str a imprimir
	    linea.append(dato['id'])
	    linea.append(dato['longitud'])
	    linea.append(dato['latitud'])
 	   linea = '|'.join(linea)
	    linea += '\n'
	    return linea

	def formatear_linea_airbnb(dato):
	    linea = []  #una lista con los str a imprimir
	    linea.append(dato['id'])
	    linea.append(dato['longitud'])
	    linea.append(dato['latitud'])
	    linea = '|'.join(linea)
	    linea += '\n'
	    return linea

	def formatear_linea_resultado(dato):
	    linea = []  #una lista con los str a imprimir
	    linea.append(str(dato['evento']))
	    linea.append(str(dato['piso']))
	    linea.append(str(dato['distancia']))
	    linea = '|'.join(linea)
	    linea += '\n'
	    return linea

	def escribir_fichero_resultado(bucked_name,fichero, datos):
 	   with tempfile.NamedTemporaryFile(delete=False, mode='w+t') as TEMPORARY_FILE:
	    	TEMPORARY_FILE.writelines(f"evento|piso|distancia\n")
	    	for dato in datos:
	        	linea= formatear_linea_resultado(dato)
	        	TEMPORARY_FILE.writelines(f"{linea}")
	    	TEMPORARY_FILE.seek(0)
 	   	upload_file_to_bucket(bucked_name, TEMPORARY_FILE, fichero)
 	   	TEMPORARY_FILE.close()
    

	
	def calcularDistancia(punto1,punto2):
		from geopy.distance import geodesic
		return geodesic(punto1, punto2).meters 

	def calcularDistancias(eventos,pisos):
		resultados=[]
		for evento in eventos:
		for piso in pisos[1:]:
	            
			punto1=(float(evento['latitud']),float(evento['longitud']))
			punto2=(float(piso['latitud']),float(piso['longitud']))

				distancia=calcularDistancia(punto1,punto2)
				resultado={ 'evento':evento['id'],
							'piso':piso['id'],
							'distancia':distancia}
				resultados.append(resultado)
		return resultados

	def activate(request):
		now = datetime.now() 
		request_json = request.get_json()
		BUCKET_NAME = 'segmento_practicas_ov'
		FILE_NAME = 'input/scrapy/conciertos.csv'
		eventos = leer_fichero(BUCKET_NAME,'input/scrapy/conciertos.csv')
		pisos = leer_fichero_airbnb(BUCKET_NAME,'input/airbnb/airbnb_depurado.csv')
		resultados=calcularDistancias(eventos,pisos)
		escribir_fichero_resultado(BUCKET_NAME,'input/distancias.csv',resultados)
		return "Succes"

El resultado se guarda también en una carpeta de nuestro segmento:

![Fichero distancias](/imagenes/distancias_sg.jpg "Fichero distancias")

Ahora que ya disponemos de todos los ficheros de entrada se procede acrear un Cluster, con los siguientes parámetros:

![Fichero cluster 1](/imagenes/c1.jpg "Fichero cluster 1")
![Fichero cluster 2](/imagenes/c2.jpg "Fichero cluster 2")
![Fichero cluster 3](/imagenes/c3.jpg "Fichero cluster 3")

Entonces como podemos ver se nos ha creado nuestro Cluster:

![Fichero cluster](/imagenes/cluster.jpg "Fichero cluster")

Ahora se procede a crear dos **Tareas**:

La primera va a crear y cargar las tablas en HIVE:

![Fichero tarea 1](/imagenes/cluster.jpg "Fichero tarea 1")

La creacion de tablas en HIVE:

	CREATE TABLE airbnb (ID INT, Listing_Url STRING, Scrape_ID STRING, Last_Scraped STRING, Name STRING, Summary STRING, Space 	STRING, Description STRING, Experiences_Offered STRING, Neighborhood_Overview STRING, Notes STRING, Transit STRING, Access STRING, Interaction STRING, House_Rules STRING, Thumbnail_Url STRING, Medium_Url STRING, Picture_Url STRING, XL_Picture_Url STRING, Host_ID STRING, Host_URL STRING, Host_Name STRING, Host_Since STRING, Host_Location STRING, Host_About STRING, Host_Response_Time STRING, Host_Response_Rate STRING, Host_Acceptance_Rate STRING, Host_Thumbnail_Url STRING, Host_Picture_Url STRING, Host_Neighbourhood STRING, Host_Listings_Count STRING, Host_Total_Listings_Count STRING, Host_Verifications STRING, Street STRING, Neighbourhood STRING, Neighbourhood_Cleansed STRING, Neighbourhood_Group_Cleansed STRING, City STRING, State STRING, Zipcode STRING, Market STRING, Smart_Location STRING, Country_Code STRING, Country STRING, Latitude STRING, Longitude STRING, Property_Type STRING, Room_Type STRING, Accommodates STRING, Bathrooms STRING, Bedrooms STRING, Beds STRING, Bed_Type STRING, Amenities STRING, Square_Feet STRING, Price FLOAT, Weekly_Price STRING, Monthly_Price STRING, Security_Deposit STRING, Cleaning_Fee STRING, Guests_Included STRING, Extra_People STRING, Minimum_Nights INT, Maximum_Nights INT, Calendar_Updated STRING, Has_Availability STRING, Availability_30 STRING, Availability_60 STRING, Availability_90 STRING, Availability_365 STRING, Calendar_last_Scraped STRING, Number_of_Reviews STRING, First_Review STRING, Last_Review STRING, Review_Scores_Rating STRING, Review_Scores_Accuracy STRING, Review_Scores_Cleanliness STRING, Review_Scores_Checkin STRING, Review_Scores_Communication STRING, Review_Scores_Location STRING, Review_Scores_Value STRING, License STRING, Jurisdiction_Names STRING, Cancellation_Policy STRING, Calculated_host_listings_count STRING, Reviews_per_Month STRING, Geolocation STRING, Features STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ';'; 
	CREATE TABLE eventos ( evento_id INT,tipo STRING, descripcion STRING, fecha_desc STRING, direccion STRING, cp STRING,zona STRING, latitud STRING, longitud STRING)  ROW FORMAT DELIMITED FIELDS TERMINATED BY '|'; 
	CREATE TABLE evento_piso ( evento_id INT, id INT, distancia FLOAT)  ROW FORMAT DELIMITED FIELDS TERMINATED BY '|';

	LOAD DATA INPATH 'gs://segmento_practicas_ov/input/airbnb/airbnb_depurado.csv' INTO TABLE airbnb;
	LOAD DATA INPATH 'gs://segmento_practicas_ov/input/scrapy/conciertos.csv' INTO TABLE eventos;
	LOAD DATA INPATH 'gs://segmento_practicas_ov/input/resultado.csv' INTO TABLE evento_piso ;

EL resultado de l cosulta

	INSERT OVERWRITE DIRECTORY 'gs://segmento_practicas_ov/output' ROW FORMAT DELIMITED FIELDS TERMINATED BY '|' 
	SELECT e.*,ep.id,ep.distancia, a.ID,a.Name, a.Street,a.Price, a.Latitude, a.Longitude, a.Bedrooms, a.Beds, a.Bathrooms, 	a.Room_Type 
	FROM eventos e 
	INNER JOIN evento_piso ep 
	ON e.evento_id=ep.evento_id 
	INNER JOIN airbnb a 
	ON ep.id=a.id 
	WHERE e.descripcion='Maldita Nerea' and ep.distancia < 1000;
