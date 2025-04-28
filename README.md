# Proyecto de Enriquecimiento de Datos: Zoila Cárdenas

Bienvenida, **D** 💛

Este documento te guiará paso a paso en la creación de un **Knowledge Graph** usando la metodología de **GraphRAG**, partiendo del dataset digitalizado del álbum de Zoila Cárdenas (https://datos.pucp.edu.pe/dataset.xhtml?persistentId=hdl:20.500.12534/BGN5UX).

Cada etapa incluye instrucciones claras y código funcional para que puedas avanzar con confianza.

---

## Paso 1: Preparación del Entorno

Antes de trabajar, es importante tener instalado **GraphDB** localmente para poder visualizar, editar y consultar el grafo que construiremos.

### Instalación de GraphDB en Windows

1. Dirígete a: [https://www.ontotext.com/products/graphdb/graphdb-free/](https://www.ontotext.com/products/graphdb/graphdb-free/)
2. Descarga la versión **GraphDB Free**.
3. Extrae el archivo ZIP descargado en una carpeta de tu elección (por ejemplo: `C:\GraphDB\`).
4. Dentro de la carpeta, ejecuta el archivo `graphdb.exe`.
5. Accede a la consola de administración a través de tu navegador en: [http://localhost:7200](http://localhost:7200)

✅ ¡Listo! Ahora tienes un servidor de GraphDB funcionando localmente.

> **Tip:** Crea un repositorio vacío llamado **zoila** para cargar los datos que construiremos.

### Instalación de librerías Python necesarias

```bash
pip install rdflib pandas requests tqdm
```

### Cargar el dataset base y hacer consultas SPARQL de prueba

Una vez que subas `dataset.ttl` a tu repositorio en GraphDB, puedes hacer consultas SPARQL para explorar los datos.

**Ejemplos de consultas:**

- Obtener todos los títulos:

```sparql
PREFIX dc: <http://purl.org/dc/elements/1.1/>

SELECT ?titulo WHERE {
  ?s dc:title ?titulo .
}
LIMIT 10
```

- Obtener todas las entidades y sus tipos:

```sparql
SELECT ?entidad ?tipo WHERE {
  ?entidad a ?tipo .
}
LIMIT 10
```

- Buscar todas las obras con algún tipo de autoría registrada:

```sparql
PREFIX dc: <http://purl.org/dc/elements/1.1/>

SELECT ?obra ?autor WHERE {
  ?obra dc:creator ?autor .
}
LIMIT 10
```

🔎 **¿Cómo hacer esto en GraphDB?**

1. Carga el archivo `dataset.ttl` en tu repositorio "zoila".
2. Dirígete a la pestaña **SPARQL** en la consola de GraphDB.
3. Copia y pega una de las consultas de ejemplo.
4. Presiona el botón **Run** y observa los resultados.

Esta exploración inicial te ayudará a familiarizarte con la estructura de los datos.

---

## Paso 2: Exploración del Dataset Base

El archivo `dataset.ttl` que has subido contiene los datos originales en formato **RDF/Turtle**. Vamos a cargarlo y explorarlo.

### Código Python para cargar el TTL

```python
from rdflib import Graph

# Carga del grafo base
g = Graph()
g.parse("/mnt/data/dataset.ttl", format="ttl")

print(f"Grafo cargado con {len(g)} triples.")

# Opcional: Mostrar algunas triples de ejemplo
for s, p, o in list(g)[:10]:
    print(s, p, o)
```

Este paso nos permite ver qué entidades, propiedades y relaciones ya existen.

---

## Paso 3: Extracción de Títulos y Enlaces a Wikipedia/Wikidata

El siguiente objetivo es buscar si los **títulos** del álbum tienen correspondencias en Wikipedia y Wikidata.

### Código Python para buscar conexiones externas

```python
import requests
from tqdm import tqdm

# Funciones auxiliares
def buscar_wikipedia(titulo, lang="es"):
    url = f"https://{lang}.wikipedia.org/w/api.php"
    params = {
        "action": "query",
        "list": "search",
        "srsearch": titulo,
        "format": "json"
    }
    response = requests.get(url, params=params)
    data = response.json()
    results = data.get("query", {}).get("search", [])
    if results:
        return f"https://{lang}.wikipedia.org/wiki/" + results[0]['title'].replace(" ", "_")
    return None

def buscar_wikidata(wikipedia_url):
    url = "https://www.wikidata.org/w/api.php"
    params = {
        "action": "wbgetentities",
        "sites": "eswiki",
        "titles": wikipedia_url.split("/")[-1],
        "format": "json"
    }
    response = requests.get(url, params=params)
    data = response.json()
    entities = data.get("entities", {})
    if entities:
        return list(entities.keys())[0]  # Q-ID
    return None

# Iterar sobre títulos del grafo
titulo_query = """
SELECT DISTINCT ?titulo WHERE {
  ?s <http://purl.org/dc/elements/1.1/title> ?titulo .
}
"""

resultados = g.query(titulo_query)

enlaces = {}

for row in tqdm(resultados, desc="Buscando enlaces externos"):
    titulo = str(row[0])
    wiki_url = buscar_wikipedia(titulo)
    if wiki_url:
        qid = buscar_wikidata(wiki_url)
        enlaces[titulo] = {"wikipedia": wiki_url, "wikidata": qid}

# Mostrar un resumen
for titulo, datos in list(enlaces.items())[:5]:
    print(f"{titulo} -> {datos}")
```

**Resultado:** Un diccionario que mapea cada título a su URL en Wikipedia y su Q-ID en Wikidata si existe.

