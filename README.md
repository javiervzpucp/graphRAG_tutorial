# Proyecto de Enriquecimiento de Datos: Zoila Cárdenas

Bienvenida, **D** 💛

Este documento te guiará paso a paso para construir un **Knowledge Graph** y un sistema tipo **GraphRAG** usando el álbum digitalizado de Zoila Cárdenas ([ver dataset](https://datos.pucp.edu.pe/dataset.xhtml?persistentId=hdl:20.500.12534/BGN5UX)).

Cada etapa incluye instrucciones claras, código funcional y buenas prácticas para que puedas aprender y construir tu propio asistente.

---

# Paso 1: Instalación de GraphDB y Primera Exploración

## 1.1 Instalación de GraphDB en Windows

1. Descarga GraphDB Free desde [Ontotext GraphDB](https://www.ontotext.com/products/graphdb/graphdb-free/).
2. Extrae el ZIP en una carpeta, por ejemplo `C:\GraphDB\`.
3. Ejecuta el archivo `graphdb.exe`.
4. Accede a la interfaz de administración en [http://localhost:7200](http://localhost:7200).

⚡ **Crea un repositorio llamado `zoila`** (tipo RDF4J - OWL2-RL Ruleset) para cargar tus datos.

---

## 1.2 Instalación de librerías Python

Asegúrate de tener instaladas las siguientes librerías necesarias:

```bash
pip install rdflib pandas requests tqdm sentence-transformers networkx node2vec
```

---

## 1.3 Cargar `dataset.ttl` y probar consultas SPARQL

Carga el archivo `dataset.ttl` en tu repositorio `zoila` y prueba estas consultas básicas en la consola de GraphDB para validar que todo esté correcto:

- **Listar títulos de las obras:**

```sparql
SELECT DISTINCT ?titulo WHERE {
  ?s <http://purl.org/dc/elements/1.1/title> ?titulo .
}
```

- **Listar todas las entidades y propiedades:**

```sparql
SELECT DISTINCT ?s ?p ?o WHERE {
  ?s ?p ?o .
} LIMIT 20
```

- **Obras y sus autores:**

```sparql
SELECT DISTINCT ?obra ?autor WHERE {
  ?obra <http://purl.org/dc/elements/1.1/creator> ?autor .
}
```

- **Obras con temas:**

```sparql
SELECT DISTINCT ?obra ?tema WHERE {
  ?obra <http://purl.org/dc/terms/subject> ?tema .
}
```

✅ **Consejo:** Explora los resultados y familiarízate con las entidades que tienes en tu grafo.

---

# Paso 2: Exploración del Dataset en Python

Ahora vamos a cargar y explorar los datos usando Python:

```python
from rdflib import Graph

g = Graph()
g.parse("dataset.ttl", format="ttl")

print(f"Grafo cargado con {len(g)} triples.")

# Mostrar ejemplos
for s, p, o in list(g)[:10]:
    print(f"{s} -- {p} --> {o}")
```

✅ **Objetivo:** Entender qué tipos de nodos y propiedades tienes en el grafo.

---

# Paso 3: Enriquecimiento Externo con Wikipedia y Wikidata

Ahora buscaremos más información en fuentes externas.

## 3.1 Buscar Enlaces Automáticamente

```python
import requests
from tqdm import tqdm

entities = set()

for s, p, o in g:
    if "title" in p or "subject" in p:
        entities.add(str(o))

def search_wikipedia(entity):
    url = "https://es.wikipedia.org/w/api.php"
    params = {
        "action": "query",
        "format": "json",
        "list": "search",
        "srsearch": entity
    }
    res = requests.get(url, params=params)
    data = res.json()
    try:
        return data["query"]["search"][0]["title"]
    except (KeyError, IndexError):
        return None

wikidata_links = {}
for entity in tqdm(entities):
    page_title = search_wikipedia(entity)
    if page_title:
        wikidata_links[entity] = f"https://es.wikipedia.org/wiki/{page_title}"

print(wikidata_links)
```

✅ **Tip:** Puedes usar otros servicios como Wikidata directamente para encontrar más propiedades (identificadores, imágenes, descripciones).

---

# Paso 4: Enriquecimiento del Grafo RDF

Vamos a agregar triples nuevos para enriquecer el grafo.

```python
from rdflib import URIRef, Literal, Namespace

schema = Namespace("http://schema.org/")

for entity, link in wikidata_links.items():
    g.add((URIRef(entity), schema.description, Literal(f"Información disponible en: {link}")))

# Guardamos el nuevo grafo enriquecido
g.serialize("dataset_enriched.ttl", format="ttl")
```

✅ **Sugerencia:** Puedes seguir añadiendo propiedades como `schema:about`, `schema:image`, etc.

---

# Paso 5: Creación y Combinación de Embeddings

Ahora crearemos embeddings combinando texto y estructura del grafo.

## 5.1 Embeddings de Texto

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

texts = []
subjects = []

for s, p, o in g:
    if "description" in p:
        texts.append(str(o))
        subjects.append(str(s))

text_embeddings = model.encode(texts)
```

✅ Cada nodo tiene ahora una representación basada en su descripción textual.

---

## 5.2 Embeddings Estructurales (Node2Vec)

```python
import networkx as nx
from node2vec import Node2Vec

G = nx.Graph()

for s, p, o in g:
    G.add_edge(str(s), str(o))

node2vec = Node2Vec(G, dimensions=64, walk_length=30, num_walks=200, workers=2)
model_n2v = node2vec.fit()

structural_embeddings = []
for subject in subjects:
    structural_embeddings.append(model_n2v.wv.get_vector(subject))
```

✅ **Recuerda:** Node2Vec captura la estructura del grafo como "caminos".

---

## 5.3 Combinación de Embeddings (60% texto + 40% estructura)

```python
import numpy as np

combined_embeddings = []

for i in range(len(subjects)):
    combined = 0.6 * text_embeddings[i] + 0.4 * structural_embeddings[i]
    combined_embeddings.append(combined)

combined_embeddings = np.array(combined_embeddings)
```

---

## 5.4 Guardado de Embeddings y Diccionario

```python
# Guardar embeddings combinados en formato .npy
np.save('combined_embeddings.npy', combined_embeddings)

# Crear diccionario entidad → embedding
import pickle

entity_to_embedding = dict(zip(subjects, combined_embeddings))

with open('entity_to_embedding.pkl', 'wb') as f:
    pickle.dump(entity_to_embedding, f)
```

✅ Ahora tienes todo listo:
- `combined_embeddings.npy`: matriz de embeddings.
- `entity_to_embedding.pkl`: diccionario de acceso rápido.

---

🎯 **Hasta aquí has construido un grafo enriquecido y representado sus entidades en vectores listos para hacer retrieval o generación.**

¡En el siguiente paso construiremos el asistente RAG! 🚀
