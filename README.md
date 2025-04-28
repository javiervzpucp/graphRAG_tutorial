# Proyecto de Enriquecimiento de Datos: Zoila Cárdenas

Bienvenida, **D** 💛

Este documento te guiará paso a paso en la creación de un **Knowledge Graph** usando la metodología de **GraphRAG**, partiendo del dataset digitalizado del álbum de Zoila Cárdenas (https://datos.pucp.edu.pe/dataset.xhtml?persistentId=hdl:20.500.12534/BGN5UX).

Cada etapa incluye instrucciones claras y código funcional para que puedas avanzar con confianza.

---

# Paso 1: Preparación del Entorno y Primeras Consultas

### Instalación de GraphDB en Windows

1. Dirígete a: [GraphDB Free](https://www.ontotext.com/products/graphdb/graphdb-free/)
2. Descarga la versión **GraphDB Free**.
3. Extrae el ZIP en `C:\GraphDB\`.
4. Ejecuta `graphdb.exe`.
5. Accede en [http://localhost:7200](http://localhost:7200).

> ⬆️ Crea un repositorio llamado **zoila** en GraphDB para cargar tu archivo `dataset.ttl`.

### Instalación de librerías Python

```bash
pip install rdflib pandas requests tqdm
```

### Cargar datos en GraphDB y ejecutar consultas SPARQL

Una vez cargado el `dataset.ttl`, prueba algunas consultas:

- **Listar títulos**

```sparql
SELECT DISTINCT ?titulo WHERE {
  ?s <http://purl.org/dc/elements/1.1/title> ?titulo .
}
```

- **Explorar entidades**

```sparql
SELECT DISTINCT ?s ?p ?o WHERE {
  ?s ?p ?o .
} LIMIT 10
```

- **Buscar autorías**

```sparql
SELECT DISTINCT ?obra ?autor WHERE {
  ?obra <http://purl.org/dc/elements/1.1/creator> ?autor .
}
```

---

# Paso 2: Exploración del Dataset Base

### Cargar y explorar el grafo en Python

```python
from rdflib import Graph

g = Graph()
g.parse("/mnt/data/dataset.ttl", format="ttl")

print(f"Grafo cargado con {len(g)} triples.")

for s, p, o in list(g)[:10]:
    print(s, p, o)
```

---

# Paso 3: Enlaces a Wikipedia y Wikidata

### Buscar enlaces externos

```python
import requests
from tqdm import tqdm

# Buscar en Wikipedia
# Buscar Q-ID en Wikidata
# Guardar enlaces en un diccionario
```

### Ejemplo de resultados

```
"Huayno" -> {"wikipedia": "https://es.wikipedia.org/wiki/Huayno", "wikidata": "Q306702"}
```

Estos enlaces permitirán **expandir** nuestro grafo original.

---

# Paso 4: Enriquecimiento Automático del Grafo

- Agregar descripciones.
- Agregar imágenes.
- Agregar categorías.

```python
# Para cada entidad conectada, consultar Wikidata y añadir propiedades
```

Resultado esperado: **+ metadatos** y **+ relaciones semánticas** para cada entidad.

---

# Paso 5: Creación de Embeddings y Combinación

## Paso 5.1: Embeddings de texto

Extraemos embeddings textuales de las descripciones enriquecidas:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

textos = []
subjects = []

for s, p, o in g.triples((None, None, None)):
    if p.endswith("wikidataDescription"):
        textos.append(str(o))
        subjects.append(str(s))

text_embeddings = model.encode(textos)
```

## Paso 5.2: Embeddings estructurales (Node2Vec)

```python
import networkx as nx
from node2vec import Node2Vec

# Crear grafo de red
G = nx.Graph()

for s, p, o in g:
    G.add_edge(str(s), str(o))

# Node2Vec
node2vec = Node2Vec(G, dimensions=64, walk_length=30, num_walks=200, workers=2)
model_n2v = node2vec.fit()

structural_embeddings = []
for subject in subjects:
    structural_embeddings.append(model_n2v.wv.get_vector(subject))
```

## Paso 5.3: Combinar embeddings

```python
import numpy as np

combined_embeddings = []

for i in range(len(subjects)):
    combined = 0.6 * text_embeddings[i] + 0.4 * structural_embeddings[i]
    combined_embeddings.append(combined)
```

---

# Esquema general del flujo de datos

```latex
\begin{tikzpicture}[node distance=1.5cm and 2cm]
\node (data) [process] {Dataset Zoila};
\node (graphdb) [process, below=of data] {GraphDB Carga + Consultas};
\node (enrich) [process, below=of graphdb] {Wikipedia/Wikidata Enriquecimiento};
\node (embedding) [process, below=of enrich] {Embeddings Textuales + Node2Vec};
\node (combined) [process, below=of embedding] {Embeddings Combinados};

\draw [arrow] (data) -- (graphdb);
\draw [arrow] (graphdb) -- (enrich);
\draw [arrow] (enrich) -- (embedding);
\draw [arrow] (embedding) -- (combined);
\end{tikzpicture}
```
