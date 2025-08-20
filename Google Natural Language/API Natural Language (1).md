# Documentación Código API Google Natural Language 
Elaborado por:  (`erickdesrosiers`)
## Tabla de Contenido 
- [Introducción](#introducción)  
- [Instalación](#instalación)
  - [Paqueterías](#paqueterías)
- [Análisis de Sentimiento](#análisis-de-sentimiento)
- [Análisis de Entidades](#análisis-de-entidades)
- [Clasificación de Contenido](#clasificación-de-contenido)
- [Moderación de Textos](#moderación-de-textos)
- [Ejemplos](#ejemplos)
---

## Introducción 
Esta documentación presenta información esencial sobre funciones construidas para la utilización de la API de `Google Natural Language`, usada en el contexto de `Webedia LATAM` para realizar clasificación de contenido, análisis de sentimiento, moderación de textos y descomposición de análisis de entidades. Esta herramienta nos permite hacer un análisis más detallado sobre el impacto y la clasificación de nuestros contenidos desde la perspectiva de `Google`.

## Instalación
Es importante, como primer paso, activar los permisos necesarios para obtener acceso a la API, descargar el archivo `.json` y guardar su ruta en un archivo `.env` al cual tengas acceso. En el siguiente video de `YouTube` se detalla con mayor precisión cómo activar el uso de esta API: [https://www.youtube.com/watch?v=iqRgOdJZtiY](https://www.youtube.com/watch?v=iqRgOdJZtiY).


### Paqueterías 
Se utilizan las siguientes paqueterías para el uso de la API. Es importante señalar que, en la sección de `Clasificación de Contenido`, es necesario importar e intalar paqueterías que nos ayuden a traducir los textos; esto se debe a que esta función únicamente funciona para el idioma inglés. Por el momento, no existe alguna manera de utilizar la `Clasificación de Contenido` sin el uso de un traductor. En esta sección también se deberán cargar las variables de entorno del archivo `.env`.

```python
pip install google-cloud-language
pip install translate
pip install google-cloud-language google-cloud-translate
```
El `client` es el "puente" entre el código y Google Cloud Natural Language API. Sin él, no se puede enviar solicitudes ni recibir resultados de análisis de texto.
```python
import pandas as pd
from translate import Translator
from google.cloud import language_v1 as language
from dotenv import load_dotenv
import os
load_dotenv()  # Carga las variables del archivo .env
credentials_path = os.getenv('GOOGLE_APPLICATION_CREDENTIALS')

os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = credentials_path
client=language.LanguageServiceClient()
```

### Análisis de Sentimiento

#### Función analizar_sentimiento_texto
La función recibe un texto en formato string, crea un cliente de la API de Google Cloud Natural Language y define el texto como un documento de tipo PLAIN_TEXT (texto sin formato).
Este envía el documento a la API para obtener el análisis de sentimiento y por último devuelve la respuesta (AnalyzeSentimentResponse).

```python
def analizar_sentimiento_texto(texto: str):
    cliente = language.LanguageServiceClient()
    documento = language.Document(
        content=texto,
        type_=language.Document.Type.PLAIN_TEXT,
    )
    respuesta = cliente.analyze_sentiment(document=documento)
    return respuesta

```
#### Función obtener_dataframes_sentimiento
Recibe la respuesta de la API, crea un DataFrame con cada oración analizada (oracion: texto de la oración, score: valor del sentimiento \[-1 negativo, +1 positivo], magnitud: intensidad del sentimiento la cual oscila entre 0 e infinito) y otro DataFrame con el resultado general del documento (puntaje: sentimiento global, magnitud: intensidad global, idioma: idioma detectado), devolviendo ambos DataFrames.

```python
def obtener_dataframes_sentimiento(respuesta: language.AnalyzeSentimentResponse):
    # DataFrame para oraciones
    oraciones = [{
        "oracion": s.text.content,
        "score": s.sentiment.score,
        "magnitud": s.sentiment.magnitude
    } for s in respuesta.sentences]
    df_oraciones = pd.DataFrame(oraciones)

    # DataFrame para documento completo
    sentimiento_doc = respuesta.document_sentiment
    df_documento = pd.DataFrame([{
        "puntaje": sentimiento_doc.score,
        "magnitud": sentimiento_doc.magnitude,
        "idioma": respuesta.language
    }])

    return df_oraciones, df_documento
```
### Análisis de Entidades

#### Función analizar_entidades_texto
La función analizar_entidades_texto recibe un texto como parámetro, crea un cliente de la API de Google Cloud Natural Language, define el texto como un documento de tipo PLAIN_TEXT, envía el documento a la API para realizar el análisis de entidades y devuelve un objeto AnalyzeEntitiesResponse con la información detectada.
```python
def analizar_entidades_texto(texto: str) -> language.AnalyzeEntitiesResponse:
    cliente = language.LanguageServiceClient()
    documento = language.Document(
        content=texto,
        type_=language.Document.Type.PLAIN_TEXT,
    )
    return cliente.analyze_entities(document=documento)
```

#### Función mostrar_entidades_texto_completo
 La función mostrar_entidades_texto_completo recibe la respuesta de la API, recorre todas las entidades encontradas en el texto y para cada una extrae sus menciones (texto y tipo), así como su nombre, tipo (persona, organización, lugar, etc.), relevancia, identificador mid y URLs asociadas de Wikipedia y Google; con estos datos crea un DataFrame y, si no está vacío, convierte la relevancia a porcentaje con un decimal, devolviendo el DataFrame final con toda la información lista para análisis o visualización.

```python
def mostrar_entidades_texto_completo(respuesta: language.AnalyzeEntitiesResponse) -> pd.DataFrame:
    datos = []
    for entidad in respuesta.entities:
        # Extraer todas las menciones (texto y tipo)
        menciones = []
        for m in entidad.mentions:
            tipo_mencion = language.EntityMention.Type(m.type_).name
            menciones.append(f"{m.text.content} ({tipo_mencion})")
        menciones_str = "; ".join(menciones)

        datos.append({
            "nombre": entidad.name,
            "tipo": language.Entity.Type(entidad.type_).name,
            "relevancia": entidad.salience,
            "mid": entidad.metadata.get("mid", ""),
            "url_wikipedia": entidad.metadata.get("wikipedia_url", ""),
            "url_google": entidad.metadata.get("google_url", ""),
            "menciones": menciones_str,
        })

    df = pd.DataFrame(datos)
    if not df.empty:
        df["relevancia"] = df["relevancia"].apply(lambda x: f"{x:.1%}")
    return df
```
### Clasificación de Contenido

#### Función traducir_texto

Esta función recibe un parámetro de entrada de tipo *string* en idioma español, define el idioma objetivo como inglés y traduce el *string*.
```python
def traducir_texto(texto: str, idioma_destino: str = "en", idioma_origen: str = "es") -> str:
    """
    Traduce el texto a un idioma objetivo usando el paquete 'translate'.
    """
    traductor = Translator(to_lang=idioma_destino, from_lang=idioma_origen)
    return traductor.translate(texto)
```

#### Función clasificar_texto
La función `clasificar_texto` recibe un texto en inglés, crea un cliente de Google Cloud Natural Language, lo define como documento de tipo PLAIN\_TEXT, solicita a la API su clasificación temática y devuelve la respuesta `ClassifyTextResponse`.

```python
def clasificar_texto(texto: str) -> language.ClassifyTextResponse:
    """
    Clasifica el texto en categorías temáticas usando Google Cloud NLP.
    Nota: El texto debe estar en inglés para que funcione.
    """
    cliente = language.LanguageServiceClient()
    documento = language.Document(
        content=texto,
        type_=language.Document.Type.PLAIN_TEXT,
        language="en"  # La API solo permite clasificación de texto en inglés
    )
    return cliente.classify_text(document=documento)
```
#### Función mostrar_clasificacion_texto
La función `mostrar_clasificacion_texto` muestra las categorías con su nivel de confianza asociado en un DataFrame.

```python
def mostrar_clasificacion_texto(respuesta: language.ClassifyTextResponse) -> pd.DataFrame:
    datos = [{
        "categoría": categoria.name,
        "confianza": f"{categoria.confidence:.0%}"
    } for categoria in respuesta.categories]

    return pd.DataFrame(datos)
```
### Moderación de Textos

La moderación de texto detecta una amplia variedad de contenidos dañinos, incluyendo discursos de odio, acoso y hostigamiento sexual. Este análisis se realiza mediante el método moderate_text, que devuelve una respuesta del tipo ModerateTextResponse.
```python
def moderar_texto(texto: str) -> language.ModerateTextResponse:
    cliente = language.LanguageServiceClient()
    documento = language.Document(
        content=texto,
        type_=language.Document.Type.PLAIN_TEXT,
    )
    return cliente.moderate_text(document=documento)
```

`Mostrar_moderacion_texto` recibe la respuesta de moderación, obtiene las categorías identificadas, las ordena por nivel de confianza de forma descendente, crea un DataFrame con el nombre de cada categoría y su confianza, y si el DataFrame no está vacío convierte la confianza a porcentaje sin decimales, devolviendo el DataFrame final con la información.

```python
def mostrar_moderacion_texto(respuesta: language.ModerateTextResponse) -> pd.DataFrame:
    categorias = respuesta.moderation_categories

    # Ordenar por confianza descendente
    categorias_ordenadas = sorted(categorias, key=lambda c: c.confidence, reverse=True)

    datos = [{
        "categoría": categoria.name,
        "confianza": categoria.confidence
    } for categoria in categorias_ordenadas]

    df = pd.DataFrame(datos)
    if not df.empty:
        df["confianza"] = df["confianza"].apply(lambda x: f"{x:.0%}")
    return df
```
---

### Ejemplos

En el siguiente ejemplo, se toma un texto de entrada de una nota publicada por Xataka MX, luego se traduce el texto, se llama a la función de clasificación y se muestra un DataFrame sobre a qué categorías pertenece el título dado como entrada. Observamos que las categorías `/Internet & Telecom/Mobile & Wireless/Mobile Phones` (71 %) y `/Computers & Electronics/Consumer Electronics` (58 %) son bastante congruentes con lo esperado.


```python
texto1 = "Pocos lo saben, pero mantener desactivado el WiFi de tu celular al salir de casa es fundamental: pasos para mejorar la seguridad de tu smartphone"

# Paso 1: Traducir el texto al inglés
texto_traducido = traducir_texto(texto1)

# Paso 2: Clasificar el texto traducido
respuesta = clasificar_texto(texto_traducido)

# Paso 3: Mostrar la clasificación en un DataFrame
mostrar_clasificacion_texto(respuesta)
categoría	confianza
0	/Internet & Telecom/Mobile & Wireless/Mobile P...	71%
1	/Computers & Electronics/Consumer Electronics	58%
```
