# acx-springboot-prices-check    

Microservicio encargado de recopilar y unificar los datos que el usuario ha configurado y que provienen de **RFR**, procesándolos y agregando la información de **Asset Control** necesaria para posteriormente extraer precios y normalizarlos.

### Diagrama de flujo


![Alt text](src/main/resources/util/acx-springboot-prices-check.svg)




###  Recepción de la certificación

El proceso comienza con una certificación que se recibe por el tópico **cib.mkrt.acx.mktrsk.certify.acplus**, dicha certificación, la recibe el microservicio y genera un ID único de ejecución, guarda la fecha de calculo y el mensaje de certificación, después de todo esto, envia la petición de inventario a RFR al tópico **cib.mkrt.rfr.mktrsk.services.request**.

A continuación se muestra un ejemplo de mensaje de petición de inventario a **RFR**.

##### Petición de inventario
```json
{
	"idFunctionality": "ES_FX_SPT_20220506122501897",
	"unit": "ES",
	"assetClass": "FX",
	"factorType": "SPT",
	"operation": "Inventory",
	"responseQueue": "cib.mkrt.acx.mktrsk.scenariomanager.riskFactors",
	"sourceSystem": "Extractor",
	"calcDate": "20220506"
}
```
</br>

### Recepción del perímetro de RFR
El microservicio al estar suscrito al tópico, debe comprobar que los mensajes que le llegan, tienen el **ID** que ha generado al recibir la certificación, para saber si se le está enviando el perímetro del inventario solicitado.
Una vez acumulado todo el batch que contienen el **ID** generado en la certificación, se realizan una serie de operaciones de guardado de información en Cassandra, que posteriormente vamos a utilizar en otros procesos, en esos procesos de guardado podemos identificar los siguientes:

 - Guardado del perímetro en la tabla **rfr_perimeter_detail**, que es una tabla donde se almacena el perímetro de **RFR** para los **targets** existentes, dicha tabla es la que se va a utilizar para luego enviar la información a la interfaz del **Scenario Manager**
 - Guardado del JSON de RFR, mensaje de certificación , ADO y Underlying correspondiente en la tabla **perimeter_data**
 - Guardado de información de datos estáticos necesarios para el **API Funcional** en la tabla **acx_underlying_symbol**

</br>

### Generación del mensaje a Airflow

El fin de este microservicio es recopilar toda la información necesaria de **Asset Control**, para generar el mensaje que desencadena una ejecución de un **DAG** en **Airflow**.
Dicho mensaje se compone de:

 - **id**
 - **unit**
 - 	**assetClass**
 - **factorType**
 - **url**

Ejemplo de mensaje de ejecución del **DAG**
 
```json
{
	"id": "ES_FX_SPT_20220510120355392",
	"unit": "ES",
	"assetClass": "FX",
	"factorType": "SPT",
	"url": "http://s3.boaw.cloudstorage.corp/scib-des-cm-mkrkdt-pocacx/extractor/ES/ES_FX_SPT_20220510120355392?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20220510T100457Z&X-Amz-SignedHeaders=host&X-Amz-Expires=172799&X-Amz-Credential=SEZPPWCKKHM4IV232KJN/20220510/eu-west-3/s3/aws4_request&X-Amz-Signature=15ca0a1e33d0a0d6b4d8030dd186c7f501322d99bc62d59d8960e28663433116"
}
```

</br>

### ¿De qué se compone el JSON de la url de  S3?

Parte del contenido del **JSON** es información del mensaje de certificación, que posteriormente necesitaremos en las ejecuciones de **Spark** (check diario, normalización..), pero también se compone de otra información que recuperamos a través de consultas al **ACX REST**, que son los siguientes:

 - Listas de **Asset Control** donde se encuentran los **ADOS** para  la unidad, assetclass y factortype de la certificación 
 - Symbols, que son una lista formada por los diferentes **ADOS** que recibimos del batch de **RFR**
 - Circuitos a los que corresponden los **ADOS** del batch
 - TreeId
 - Calendario
 - Las dos últimas fechas de certificación (cert date y prev cert date)
 - Los campos de precio de **Asset Control** para los **ADOS** del batch (C0#CLOSE, C0#CLOSE_MTX2..)
 - La fecha de consolidación de los circuitos
 - Atributos estáticos 

Ejemplo de **JSON** de la url de  **S3** 

```json
{
	"id": "ES_FX_SPT_20220510120355392",
	"unit": "ES",
	"factorType": "SPT",
	"assetClass": "FX",
	"endDate": "2022-02-23",
	"operation": "certifySPT",
	"searchFilter": "",
	"fieldName": "",
	"returnsFields": "",
	"infoType": "",
	"responseQueue": "",
	"sourceSystem": "AC",
	"targetSystem": "",
	"startDate": "2022-02-23",
	"numScen": "",
	"runType": "",
	"calcDate": "2022-02-23",
	"treeId": "EOD_ES",
	"calendar": "SAN_AIRE",
	"circuits": [{
		"symbols": ["C0.GBMRISK.1000000500015", "C0.GBMRISK.1000000500042", "C0.GBMRISK.1000000500132", "C0.GBMRISK.1000000500165", "C0.GBMRISK.1000000500199"],
		"serviceId": "MDFX000001",
		"list": "LST.1000000116",
		"serviceIdField": "C0#SERVICE_ID",
		"certDate": "2022-02-23T14:01:04Z",
		"previousCert": "2022-02-22T14:01:04Z",
		"consolidationDate": "2022-02-23T13:45:00Z",
		"acxStaticAttributtes": {
			"CURRENCY": "C0#MD_CURRENCY",
			"longname": "prop.longname",
			"underlying": "C0#ACX_API_FUNCT",
			"BASE_CURRENCY": "C0#MD_BASE_CURRENCY"
		},
		"fields": [{
			"priceField": "Price",
			"priceType": "C0#CLOSE"
		}]
	}]
}
```

Una vez generado el mensaje mostrado anteriormente, se valida que la información esté completa (que existen fechas de certificación, fecha de consolidación, circuito... etc), si la información es correcta, se ejecuta el **DAG** **CHECK_ACX_DATA**, cuyo parámetro de ejecución es el mensaje mostrado anteriormente, que contiene la url de **S3**, una vez ejecutado el **DAG**, se borra dicho **ID** de la memoria del microservicio y cualquier batch que llegue con ese **ID**, al no existir, se omitirá.

</br>

### Funcionalidades extra del microservicio
#### Endpoints

 - **/consumption** method **POST**: a este endpoint le llega la información del consumo de un extractor determinado, este endpoint es el encargado de generar el JSON que se envía por el tópico **cib.mkrt.acx.mktrsk.api.spark.response** con dicha información
 
	 **Body**

	    {
	    	"id_functionality": "ES_FX_SPT_20220513134223801",
	    	"identificadorS3": "ES_FX_SPT_20220513134223801_CRISIL_MARKET_DATA_CONSUMPTION.csv",
	    	"santanderClient": "CRISIL",
	    	"start_date": "2007-01-01T00:00:00.000Z",
	    	"end_date": "2022-04-01T23:59:59.000Z",
	    	"requestTime": "2022-05-13T11:45:54.000Z",
	    	"responseTime": "2022-05-13T11:45:59.000Z"
	    }
 - **/restart** method **GET**: reinicia el microservicio manteniendo los **LOGS** actuales
 - **/deleteCache** method **DELETE**: borra la caché del microservicio (batches acumulados, ids de solicitud de perimetro)
 - **/createIdFunc** method **POST**: necesario hacer la petición con un mensaje de certificación, genera un ID de ejecución valido sin pedir inventario a RFR
 
	**Body:**

	```json
	{
		"unit": "ES",
		"factorType": "SPT",
		"assetClass": "FX",
		"endDate": "20220223",
		"operation": "certifySPT",
		"searchFilter": "",
		"fieldName": "",
		"returnsFields": "",
		"infoType": "",
		"responseQueue": "",
		"sourceSystem": "AC",
		"targetSystem": "",
		"startDate": "",
		"numScen": "",
		"runType": "",
		"calcDate": "20220223"
	}
	```

 - **/cacheInfo** method **GET**: devuelve la información de los IDs, las certificaciones y fechas a la espera de actualizar el perímetro
  - **/deleteCacheRequest** method **POST**: necesario hacer la petición con el ID que se quiera eliminar de la recarga de perímetro y los batches actuales acumulados
 
	**Request Parameter:**  
	
	    /deleteCacheRequest?id=ES_FX_SPT_20220511105534600
  - **/RfrPerimeterDetail/Target/remove** method **POST**: elimina un target de la tabla **rfr_perimeter_detail**
 
	 **Body**

	    {
	    	"target": "CRISIL"
	    }

 - **/RfrPerimeterDetail/Target/add** method **POST**: añade un target a la tabla rfr_perimeter_detail
 
	 **Body**
	  

	    {
		    "target": "CRISIL"
	    }

 - **/RfrPerimeterData/certify** method **POST**: avisa al microservicio de que ha finalizado la normalización, y este mismo, notifica por el tópico  **cib.mkrt.acx.mktrsk.certify.acplus** de que el dato normalizado ya esta disponible
 
	 **Body**

	    {
		    "id":  "ES_FX_SPT_20220425120539768"
	    }

</br>

### Tablas

 - **rfr_perimeter_detail**
 
 
| target | unit | assetclass | factortype | underlying | checktofile | end_date | start_date | status |
	|--------|------|------------|------------|------------|-------------|----------|------------|--------|
	| CRISIL | ES   | FX         | SPT        | EUR-USD    | true        |          |            |        |

---
 - **perimeter_data**
 
 
| id_functionality            | unit | assetclass | factortype | underlying | symbol                   | cert_json                                      | value                                          |
	|-----------------------------|------|------------|------------|------------|--------------------------|------------------------------------------------|------------------------------------------------|
	| ES_FX_SPT_20220506132737208 | ES   | FX         | SPT        | EUR-USD    | C0.GBMRISK.1000000500132 | java.nio.HeapByteBuffer[pos=0 lim=292 cap=292] | java.nio.HeapByteBuffer[pos=0 lim=299 cap=299] |

---
 - **acx_underlying_symbol**
   
| unit | asset_class | factor_type | underlying | symbol_array               |
	|------|-------------|-------------|------------|----------------------------|
	| ES   | FX          | SPT         | EUR-USD    | [C0.GBMRISK.1000000500132] |


---


