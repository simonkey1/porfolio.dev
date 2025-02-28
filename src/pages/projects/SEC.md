---
layout: ../../layouts/MarkdownPostLayout.astro
title: "Análisis en Tiempo Real del Suministro Eléctrico de Chile"
pubDate: 2025-02-12
description: "Un tutorial de cómo extraer datos con playwright, procesarlos con pandas y visualizar en Powerbi."
author: "Simon Gomez"
image:
  url: "/Imagen_amarillo1linkedin.png"
  alt: "The Astro logo on a dark background with a pink glow."
tags: ["Powerbi", "Python", "Web Scraping"]
---

## Introducción

Hace poco descubrí la página de la Superintendecia de Electricidad y Combustibles (SEC). Me llamó la atención en particular su sección de métricas e información de los cortes de luz a nivel nacional, debido a que noté que hay un gran margen de mejora en su accesibilidad y visualización de los datos para el usuario.

## ¿Qué haremos?

1. **Preparar nuestro entorno**: primero, instalaremos las dependencias necesarias para el proyecto.

2. **Extracción de los datos**: aprenderemos a realizar web scraping con Playwright.

3. **Transformación de los datos**: necesitaremos limpiar y procesar los datos extraídos con Pandas.

4. **Visualización de los datos**: finalmente, visualizaremos los datos en Powerbi.

## Entorno de Desarrollo

Para este proyecto, necesitaremos instalar las siguientes dependencias: Playwright, Pandas y Powerbi.

Les recomiendo crear un entorno virtual para instalar las dependencias. Para ello, decidan un directorio donde quieran crear el entorno virtual y ejecuten el siguiente comando en la terminal:

```python

        python -m venv "nombre-del-entorno"-env


```

Una vez creado el entorno virtual, actívenlo con el siguiente comando:

```bash

     source "nombre-del-entorno"-env/bin/activate


```

Ahora, instalemos las dependencias necesarias:

```bash

    pip install playwright pandas

```

## Extracción de los Datos

Primero, necesitamos importar las librerías necesarias:

```python

    from playwright.sync_api import sync_playwright
    import pandas as pd

```

Una vez creado el entorno virtual, actívalo:

bash

source "nombre-del-entorno"-env/bin/activate

# 🚀 Instalar Dependencias

Con el entorno virtual activado, instalemos las librerías necesarias:

```bash
pip install playwright pandas
```

Además, debemos instalar los navegadores de Playwright:

```bash
playwright install
```

---

## 🔍 Extracción de Datos con Playwright

Usaremos **Playwright** para interceptar las respuestas de la API de la SEC.

### 📌 Importación de Librerías

```python
from playwright.sync_api import sync_playwright
import pandas as pd
import os
import time
import re
from datetime import datetime, timedelta
```

---

## 📂 Definición de Archivos de Salida

Guardaremos los datos en dos archivos CSV:

- **`clientes_afectados_tiempo_real.csv`**: Contiene los datos más recientes.
- **`clientes_afectados_historico.csv`**: Mantiene un registro histórico.

```python
csv_tiempo_real = "clientes_afectados_tiempo_real.csv"
csv_historico = "clientes_afectados_historico.csv"
```

---

# 🔎 Análisis de la Función `intercept_responses()`

Esta función usa **Playwright** para interceptar las respuestas de la API de la **Superintendencia de Electricidad y Combustibles de Chile (SEC)** y extraer información sobre cortes de luz. Luego, almacena estos datos en archivos CSV.

---

## 📌 **Estructura General**

1. **Abrir un navegador en modo headless** (sin interfaz gráfica).
2. **Interceptar las respuestas de la API** en la página de la SEC.
3. **Extraer información clave** de los datos JSON recibidos.
4. **Procesar los datos** para calcular tiempos y crear identificadores únicos.
5. **Guardar la información en archivos CSV**.
6. **Cerrar el navegador** una vez completado el proceso.

---

## 🛠 **Paso a Paso de la Función**

### 1️⃣ **Inicializar Playwright y Abrir el Navegador**

Se utiliza `sync_playwright()` para iniciar Playwright y lanzar un navegador **Chromium** en modo **headless** (sin interfaz gráfica).

```python
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)  # Lanzar navegador en modo headless
    page = browser.new_page()
```

📌 **¿Qué hace esto?**

- Crea una instancia de Playwright.
- Lanza un navegador Chromium sin interfaz visual.
- Crea una nueva página en ese navegador.

---

### 2️⃣ **Definir una Lista para Almacenar Registros**

```python
registros = []  # Lista para almacenar datos nuevos
```

📌 **¿Para qué sirve?**

- Aquí se guardarán los datos extraídos de la API antes de escribirlos en los archivos CSV.

---

### 3️⃣ **Interceptar las Respuestas de la API**

```python
def handle_response(response):
    if "GetPorFecha" in response.url:
        try:
            data = response.json()
            timestamp_actual = datetime.now()  # Tiempo de consulta
```

📌 **¿Qué hace esto?**

- **Verifica si la URL de la respuesta contiene `"GetPorFecha"`**, lo que indica que es una respuesta de la API relevante.
- **Convierte la respuesta en JSON** (`response.json()`).
- **Guarda el timestamp actual** para identificar cuándo se hizo la consulta.

---

### 4️⃣ **Procesar los Datos Extraídos**

```python
for entry in data:
    actualizado_hace = entry.get("ACTUALIZADO_HACE", "")
    minutos_atras = 0

    match = re.search(r'(\d+)\s+Minutos', actualizado_hace)
    if match:
        minutos_atras = int(match.group(1))  # Extrae el número antes de "Minutos"

    hora_exacta_reporte = timestamp_actual - timedelta(minutes=minutos_atras)
```

📌 **¿Qué hace esto?**

- **Extrae el campo `"ACTUALIZADO_HACE"`**, que indica hace cuánto tiempo se actualizó la información.
- **Usa una expresión regular (`re.search`) para extraer los minutos** mencionados en `"ACTUALIZADO_HACE"`.
- **Calcula la hora exacta del reporte**, restando esos minutos del timestamp actual.

---

### 5️⃣ **Crear un Identificador Único para Cada Registro**

```python
unique_id = f"{entry.get('FECHA_INT_STR', '')}-{entry.get('REGION', '')}-{entry.get('COMUNA', '')}-{entry.get('EMPRESA', '')}-{entry.get('CLIENTES_AFECTADOS', 0)}-{hora_exacta_reporte.strftime('%Y-%m-%d %H:%M:%S')}"
```

📌 **¿Por qué es importante esto?**

- **Evita la duplicación de datos**, asegurando que cada registro tenga un ID único.
- **Facilita la organización** en los archivos CSV.

---

### 6️⃣ **Guardar los Datos en la Lista `registros`**

```python
registros.append({
    "ID_UNICO": unique_id,
    "TIMESTAMP": timestamp_actual.strftime("%Y-%m-%d %H:%M:%S"),
    "HORA_EXACTA_REPORTE": hora_exacta_reporte.strftime("%Y-%m-%d %H:%M:%S"),
    "FECHA": entry.get("FECHA_INT_STR", ""),
    "REGION": entry.get("NOMBRE_REGION", ""),
    "COMUNA": entry.get("NOMBRE_COMUNA", ""),
    "EMPRESA": entry.get("NOMBRE_EMPRESA", ""),
    "CLIENTES_AFECTADOS": entry.get("CLIENTES_AFECTADOS", 0),
    "ACTUALIZADO_HACE": actualizado_hace
})
```

📌 **¿Qué hace esto?**

- **Guarda cada registro como un diccionario** dentro de la lista `registros`.
- **Almacena los datos clave** como fecha, región, comuna, empresa y clientes afectados.

---

### 7️⃣ **Capturar las Respuestas de la API**

```python
page.on("response", handle_response)
```

📌 **¿Qué hace esto?**

- **Asocia la función `handle_response` con cada respuesta de la página**.
- **Intercepta las respuestas en segundo plano** mientras se carga la web.

---

### 8️⃣ **Acceder a la Página de la SEC**

```python
page.goto("https://apps.sec.cl/INTONLINEv1/index.aspx")
page.wait_for_timeout(5000)  # Espera para capturar datos
```

📌 **¿Qué hace esto?**

- **Abre la página de la SEC en el navegador**.
- **Espera 5 segundos** para permitir la carga de datos.

---

### 9️⃣ **Cerrar el Navegador**

```python
browser.close()
```

📌 **¿Por qué es importante?**

- **Libera recursos del sistema**.
- **Evita que el script consuma demasiada memoria**.

---

## 📊 **Guardado de Datos en CSV**

```python
if registros:
    df_new = pd.DataFrame(registros)

    # 📌 Guardar en CSV histórico
    if os.path.exists(csv_historico):
        df_historico = pd.read_csv(csv_historico, encoding="utf-8-sig")
        df_historico = pd.concat([df_historico, df_new]).drop_duplicates(subset=["ID_UNICO"], keep="first")
    else:
        df_historico = df_new

    df_historico.to_csv(csv_historico, index=False, encoding="utf-8-sig")

    # 📌 Guardar en CSV de Tiempo Real
    df_new.to_csv(csv_tiempo_real, index=False, encoding="utf-8-sig")

    print(f"✅ Datos guardados en:\n📌 {csv_historico} (Histórico)\n📌 {csv_tiempo_real} (Tiempo Real)")
```

📌 **¿Qué hace esto?**

- **Convierte los registros en un DataFrame de Pandas**.
- **Guarda los datos en CSV histórico y de tiempo real**.
- **Evita duplicados basándose en el ID único**.

---

## 🔁 **Automatización Cada 5 Minutos**

```python
while True:
    intercept_responses()
    print("⏳ Esperando 5 minutos para la siguiente ejecución...\n")
    time.sleep(5 * 60)  # 5 minutos en segundos
```

📌 **¿Qué hace esto?**

- **Ejecuta `intercept_responses()` en un bucle infinito**.
- **Espera 5 minutos (`time.sleep(5 * 60)`) antes de volver a ejecutar la función**.

---

✅ **¡Con esto, el script puede monitorear los cortes de luz en tiempo real y guardarlos en CSV de manera automática!** 🚀
