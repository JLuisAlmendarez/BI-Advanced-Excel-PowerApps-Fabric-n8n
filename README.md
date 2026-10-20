# BI de punta a punta: Olist E-commerce

**Excel · Microsoft Fabric · Power BI · SQL · Power Automate · Power Apps · n8n · Claude**

Sistema de inteligencia de negocios construido sobre el [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce): unos 100 mil pedidos reales de 2016 a 2018, repartidos en 9 tablas relacionadas.

La pregunta de negocio que guía todo el proyecto es:

> **¿Por qué algunos clientes califican mal sus pedidos y qué vendedores y categorías generan más retrasos?**

El proyecto avanza en cuatro partes: primero se **explora** el problema en Excel, luego se **mide** con un reporte en Power BI sobre Fabric, después se **actúa** con una app donde cada vendedor atiende sus malas reseñas, y al final se **explica** con un analista automático que usa Claude para responder preguntas sobre los datos.

---

## Contenido

1. [Exploración en Excel](#1-exploración-en-excel)
2. [Lakehouse en Fabric y reporte en Power BI](#2-lakehouse-en-fabric-y-reporte-en-power-bi)
3. [Portal del vendedor: chat con el cliente](#3-portal-del-vendedor-chat-con-el-cliente)
4. [Analista con IA: n8n + Claude](#4-analista-con-ia-n8n--claude)
5. [Hallazgos](#5-hallazgos)
6. [Problemas resueltos en el camino](#6-problemas-resueltos-en-el-camino)
7. [Estructura del repositorio](#7-estructura-del-repositorio)

---

## Herramientas

| Herramienta | Rol en el proyecto |
|---|---|
| **Excel** | Exploración y prototipo del tablero (Power Query, Power Pivot, tablas dinámicas) |
| **Fabric** | Lakehouse con capas bronze → silver → gold, Dataflows Gen2, notebooks y pipelines |
| **Power BI** | Modelo estrella en Direct Lake, medidas DAX y reporte ejecutivo |
| **SQL database (Fabric)** | Tablas de seguimiento de quejas que la app y los flujos pueden escribir |
| **Power Automate** | Avisos a vendedores y conversación por correo con el cliente |
| **Power Apps** | Portal donde el vendedor responde y conversa con el cliente |
| **n8n** | Tipo de cambio diario y orquestación del analista con IA |
| **Claude (API)** | Analista que lee los KPIs del modelo y redacta el diagnóstico |

---

## 1. Exploración en Excel

**Objetivo:** entender los datos y prototipar el tablero antes de construir la solución en la nube.

- **Power Query:** limpieza de las 9 tablas, con tipos de fecha corregidos, duplicados eliminados y una revisión de fechas inconsistentes (por ejemplo, una entrega anterior a la compra). Se agregaron columnas calculadas: `delivery_days`, `delay_days` e `is_late_delivery`.
- **Power Pivot:** modelo de datos relacionado y medidas básicas (ventas, pedidos, calificación promedio).
- **Tablero prototipo:** KPIs, segmentadores por fecha y estado, y un gráfico por cada pregunta de negocio.

![Prototipo en Excel](Appx/Excel/imgs/Dashbord%20Excel%20Prototype.png)

<details>
<summary>Más capturas de Power Query</summary>

![Esquema de Power Query](Appx/Excel/imgs/Excel%20Power%20Query%20Schema.png)
![Secuencia de Power Query 1](Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2001.png)
![Secuencia de Power Query 2](Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2002.png)
![Tabla dinámica](Appx/Excel/imgs/Other%20dynamic%20Table.png)

</details>

📄 Código M: [`Appx/Excel/excel_power_query.txt`](Appx/Excel/excel_power_query.txt) · 📊 Libro: [`analysis/Analysis E-commerce Dashboard Prototype.xlsx`](analysis/) · 🐍 EDA en Python: [`analysis/data_eda.ipynb`](analysis/data_eda.ipynb)

---

## 2. Lakehouse en Fabric y reporte en Power BI

**Objetivo:** llevar el prototipo a una arquitectura medallion en Microsoft Fabric y publicar el reporte ejecutivo.

**Lakehouse (medallion)**
- **Bronze:** los 9 CSV tal como vienen de Kaggle, en `Files/bronze/`.
- **Silver:** limpieza con **Dataflows Gen2**, reutilizando la misma lógica de Power Query del prototipo en Excel.
- **Gold:** tablas Delta listas para el modelo, incluida una tabla `calendar`.

![Tablas bronze](Appx/Fabric/imgs/Lakehouse%20Raw%20Tables%20(Bronze).png)
![Tablas gold](Appx/Fabric/imgs/Lakehouse%20Processed%20Tables%20(Gold).png)

**Modelo semántico**
- **Modo:** Direct Lake sobre la capa gold.
- **Relaciones:** varios a uno con filtro de dirección única. Las medidas que necesitan filtrar "hacia arriba" usan `CROSSFILTER` solo dentro de su cálculo.
- **Medidas DAX:** ventas, ticket promedio, clientes únicos (con `customer_unique_id`), % de entregas tarde, días de entrega, calificación promedio y, en la parte 4, **Ventas USD**.

![Modelo semántico](Appx/Fabric/imgs/Semantic%20Model%20Schema.png)

**Reporte ejecutivo**

![Reporte en Power BI](Appx/Fabric/imgs/Dashboard%20Power%20BI.png)

<details>
<summary>Más capturas del Dataflow</summary>

![Ejecución del Dataflow (silver)](Appx/Fabric/imgs/DataFlow%20Execution%20(Silver).png)
![Esquema de Power Query en Fabric](Appx/Fabric/imgs/Fabric%20Power%20Query%20Schema.png)
![Secuencia 1](Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2001.png)
![Secuencia 2](Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2002.png)
![Secuencia 3](Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2003.png)

</details>

📄 Medidas y relaciones: [`Appx/Fabric/measures_dax.txt`](Appx/Fabric/measures_dax.txt) · 📄 Dataflow: [`Appx/Fabric/dataflow_fabric_power_query.txt`](Appx/Fabric/dataflow_fabric_power_query.txt) · 📑 Reporte: [`Appx/Fabric/Reporte.pdf`](Appx/Fabric/Reporte.pdf)

---

## 3. Portal del vendedor: chat con el cliente

**Objetivo:** pasar de *medir* las malas reseñas a *atenderlas*. Cada vez que un vendedor recibe una reseña de 1 o 2 estrellas, le llega un aviso automático y puede contactar al cliente desde una app.

### 3.1 Tablas de seguimiento (SQL database en Fabric)

El SQL endpoint del Lakehouse es de **solo lectura**, así que la app y los flujos no podrían escribir ahí. Por eso se creó una **SQL database de Fabric** con tres tablas:

| Tabla | Contenido |
|---|---|
| `vendedor_contacto` | Vendedores de la demo y su correo |
| `seguimiento_resenas` | Una fila por reseña de 1–2★ y vendedor, con estado, respuesta y fechas |
| `interacciones` | Los mensajes del chat entre vendedor y cliente |

Una queja pasa por estos estados: `sin_respuesta` → `en_contacto` → `cerrada`.

Los datos se cargan con un **Copy job** del Lakehouse a tablas de staging, y luego se filtran y relacionan en T-SQL. También hay **procedimientos almacenados**, para que Power Automate no tenga que armar consultas.

![Copia del Lakehouse a SQL](Appx/Fabric/imgs/copia_tablas_lakehouse_a_sql.png)
![Tablas y procedimientos](Appx/Fabric/imgs/tablas_y_procedimientos_seguimientos.png)

> **Datos simulados:** Olist no trae correos de vendedores ni de clientes. Se usaron los 5 vendedores con más malas reseñas (1,376 quejas en total) y alias con `+` de una sola cuenta de correo, como `cuenta+vendedor1@gmail.com`.

📄 [`tablas_seguimiento_sql.txt`](Appx/Fabric/tablas_seguimiento_sql.txt) · 📄 [`procedimientos_tablas_seguimiento_sql.txt`](Appx/Fabric/procedimientos_tablas_seguimiento_sql.txt)

### 3.2 Avisos y correos (Power Automate)

| Flujo | Disparador | Qué hace |
|---|---|---|
| **Aviso a vendedores** | Programado | Manda un correo por vendedor con sus quejas pendientes y la liga a la app, y las marca como avisadas |
| **Respuesta del cliente** | Correo entrante con `[Queja #` en el asunto | Saca el número de queja del asunto y guarda la respuesta en `interacciones` |

![Flujos de Power Automate](Appx/Automate/imgs/lista_flujos_automate.png)
![Aviso a vendedores](Appx/Automate/imgs/avisos_a_vendedores.png)
![Respuestas del cliente](Appx/Automate/imgs/respuestas_del_cliente.png)

<details>
<summary>Más capturas</summary>

![De la base al correo del vendedor](Appx/Automate/imgs/db_seguimiento_a_correo_vendedor.png)
![Del correo a la tabla de interacciones](Appx/Automate/imgs/correo_a_db_interacciones.png)

</details>

📄 Expresiones de los flujos y errores resueltos: [`Appx/Automate/fx_expressions_used_in_flows.txt`](Appx/Automate/fx_expressions_used_in_flows.txt)

### 3.3 App del vendedor (Power Apps)

Canvas app con tres pantallas:
- **Lista:** las quejas del vendedor, con filtro por estado.
- **Detalle:** la queja completa y el botón para responder al cliente.
- **Chat:** la conversación por correo, vista como un chat, y el botón para cerrar la queja.

| Lista de quejas | Responder | Chat |
|---|---|---|
| ![Lista sin respuesta](Appx/PowerApps/lista_sin_respuesta.png) | ![Detalle](Appx/PowerApps/detalle_responder.png) | ![Chat con respuesta del cliente](Appx/PowerApps/chat_respuesta_cliente.png) |

**El ciclo completo, visto desde los tres lados:**

| El cliente (Gmail) | La bandeja que lee Power Automate | La base de datos |
|---|---|---|
| ![Gmail](Appx/PowerApps/gmail_conversacion_cliente.png) | ![Outlook](Appx/PowerApps/outlook_respuesta_cliente.png) | ![Conversación en SQL](Appx/PowerApps/sql_conversacion_queja.png) |

<details>
<summary>Más capturas</summary>

![Primer mensaje del vendedor](Appx/PowerApps/chat_mensaje_vendedor.png)
![Lista en contacto](Appx/PowerApps/lista_en_contacto.png)
![Lista cerrada](Appx/PowerApps/lista_cerrada.png)
![Resumen por vendedor](Appx/PowerApps/sql_resumen_por_vendedor.png)
![Quejas atendidas](Appx/PowerApps/sql_quejas_atendidas.png)

</details>

---

## 4. Analista con IA: n8n + Claude

**Objetivo:** que cualquier persona pueda hacerle preguntas al reporte en lenguaje natural y reciba un diagnóstico basado en los datos del modelo.

### 4.1 Programa 1: tipo de cambio diario

Las ventas de Olist están en reales (BRL). Para mostrarlas en dólares, n8n trae la tasa oficial **PTAX** del Banco Central do Brasil.

```
API del Banco Central (11:00) → n8n → Google Sheets → notebook (12:00) → Lakehouse → modelo Direct Lake
```

- **Carga histórica:** se corrió una sola vez y trajo 750 tasas diarias de 2016 a 2018.
- **Carga diaria:** corre de lunes a viernes y agrega la tasa del día.
- **Por qué pasa por un notebook:** un modelo en Direct Lake solo lee tablas de OneLake, así que el notebook `nb_tipo_cambio` copia la hoja al Lakehouse.
- **Medida `Ventas USD`:** divide las ventas de cada día entre la tasa de ese día. En fines de semana y días festivos usa la tasa del último día hábil.

| Carga histórica | Carga diaria |
|---|---|
| ![n8n carga histórica](Appx/n8n/Program1/imgs/n8n_carga_historica.png) | ![n8n diario](Appx/n8n/Program1/imgs/n8n_diario.png) |

![Ventas en BRL y USD por año](Appx/n8n/Program1/imgs/dax_ventas_usd.png)

<details>
<summary>Más capturas</summary>

![Hoja de Google Sheets](Appx/n8n/Program1/imgs/sheets_tipo_cambio.png)
![Notebook](Appx/n8n/Program1/imgs/fabric_notebook.png)
![Programación del notebook](Appx/n8n/Program1/imgs/notebook_programacion.png)
![Modelo con tipo_cambio](Appx/n8n/Program1/imgs/modelo_con_tipo_cambio.png)

</details>

📄 [`MedidaDAX.txt`](Appx/n8n/Program1/MedidaDAX.txt) · ⚙️ [`Tipo_cambio_carga_historica.json`](Appx/n8n/Program1/Tipo_cambio_carga_historica.json) · ⚙️ [`Tipo_cambio_diario.json`](Appx/n8n/Program1/Tipo_cambio_diario.json) · 📓 [`nb_tipo_cambio.html`](Appx/n8n/Program1/nb_tipo_cambio.html)

### 4.2 Programa 2: analista con Claude

**El reto.** La idea original era que n8n consultara Power BI con su API REST, pero eso requiere registrar una aplicación en Microsoft Entra, y el tenant de la universidad lo tiene bloqueado.

**La solución.** El que lee el modelo es un **notebook de Fabric**, que corre con la sesión del usuario y no necesita permisos extra. Usa **Semantic Link** (`sempy`) para ejecutar DAX contra el modelo semántico y le **empuja** los datos a n8n por un webhook protegido con una clave en el encabezado.

**1. El cubo de KPIs.** El notebook no manda porcentajes ya calculados, sino sus **componentes** (ventas, órdenes, entregas tarde, entregas totales, suma de calificaciones, reseñas…), en dos tablas: *estado × mes* y *categoría × mes*. Así n8n puede recalcular cualquier KPI para cualquier combinación de filtros sin errores. Promediar porcentajes ya calculados daría resultados incorrectos.

![Notebook nb_kpis](Appx/n8n/Program2/imgs/fabric_nb_kpis.png)

**2. Siempre al día.** El pipeline `pl_diario` corre `nb_tipo_cambio` y luego `nb_kpis`, así que el cubo nunca es más viejo que el modelo. Cada respuesta incluye la fecha de corte de los datos.

| Programación | Ejecución |
|---|---|
| ![Programación del pipeline](Appx/n8n/Program2/imgs/fabric_pl_diario_programacion.png) | ![Ejecución del pipeline](Appx/n8n/Program2/imgs/fabric_pl_diario_ejecucion.png) |

![Recepción del cubo en n8n](Appx/n8n/Program2/imgs/n8n_recepcion_cubo.png)

**3. La consulta.** El usuario llena un formulario: estado, año, categoría, KPIs, tipo de análisis (resumen ejecutivo, comparación contra el año anterior o detección de problemas), una pregunta libre y su correo. n8n filtra el cubo, recalcula los KPIs elegidos y arma el contexto para Claude, junto con una liga al reporte de Power BI **ya filtrado**.

![Workflow de consulta](Appx/n8n/Program2/imgs/n8n_analista_consulta.png)

> El nodo del modelo aparece desactivado en la captura y en el JSON porque se retiró la API key antes de exportar.

**4. La respuesta.** Claude recibe instrucciones de usar **solo** los datos del JSON y de no inventar causas ni cifras. En la prueba, la pregunta fue *"¿Por qué bajó la calificación en São Paulo?"*. Claude **contradijo la premisa**: la calificación anual de SP subió de 4.15 a 4.19. Luego señaló la caída real en febrero y marzo de 2018 y la relacionó con el aumento de entregas tarde.

| Formulario | Respuesta por correo |
|---|---|
| ![Formulario](Appx/n8n/Program2/imgs/formulario_analista.png) | ![Correo con el análisis](Appx/n8n/Program2/imgs/correo_analisis_claude.png) |

⚙️ [`Analista_recepcion_cubo.json`](Appx/n8n/Program2/Analista_recepcion_cubo.json) · ⚙️ [`Analista_consulta.json`](Appx/n8n/Program2/Analista_consulta.json) · 📓 [`nb_kpis.html`](Appx/n8n/Program2/nb_kpis.html)

---

## 5. Hallazgos

1. **El retraso destruye la satisfacción.** Los pedidos a tiempo promedian alrededor de **4.3 estrellas**; los que llegan de 6 a 15 días tarde caen a cerca de **1.8**.
2. **La puntualidad empeoró al crecer.** Las ventas pasaron de R$ 6.16 M en 2017 a R$ 7.39 M en 2018 (de enero a agosto), mientras el % de entregas tarde subió de **6.6% a 9.3%**.
3. **Febrero y marzo de 2018 fueron la crisis.** A nivel nacional, las entregas tarde llegaron a **16% y 21.4%**, y la calificación bajó a **3.83 y 3.75**, la más baja del periodo.
4. **El problema es geográfico.** Los estados del norte y noreste concentran los retrasos: Alagoas (**23.2%** en 2018), Maranhão y Ceará superan el 20%.
5. **Una categoría destaca por mala.** `office_furniture` es la peor calificada en 2018, con **3.61** de promedio y **23.6%** de reseñas negativas.
6. **El tipo de cambio cambia la lectura.** Medido en dólares, el crecimiento de 2018 es menor, porque el real se debilitó de ~3.20 a ~3.50 por dólar.

---

## 6. Problemas resueltos en el camino

| Problema | Solución |
|---|---|
| El SQL endpoint del Lakehouse es de solo lectura y la app no podía escribir | SQL database de Fabric para el seguimiento |
| El conector de SQL es premium en Power Apps y Power Automate | Entorno del Power Apps Developer Plan |
| El flujo de respuestas se disparaba con correos ajenos (Zoom, notificaciones de error) y causaba un ciclo de fallas | Filtros por remitente y por asunto en el disparador |
| El correo institucional agrega un aviso de "Precaución" a todo correo externo | Documentado como limitación de la demo |
| El registro de aplicaciones en Entra está bloqueado, así que no hay acceso a la API de Power BI | Notebook con Semantic Link que empuja los datos a n8n por webhook |
| Direct Lake no puede leer Google Sheets | Notebook que copia la hoja al Lakehouse |
| Los porcentajes no se pueden reagregar con filtros distintos | Cubo con componentes; los KPIs se recalculan en n8n |
| Claude respondía en markdown y el correo se veía como texto plano | Conversión de markdown a HTML con soporte de tablas |

---

## 7. Estructura del repositorio

```
├── kaggle_data/           # 9 CSV originales de Olist
├── analysis/              # Prototipo en Excel, EDA en Python y reporte en PDF
└── Appx/
    ├── Excel/             # Power Query del prototipo y capturas
    ├── Fabric/            # Dataflow, medidas DAX, SQL de seguimiento y capturas
    ├── Automate/          # Expresiones de los flujos y capturas
    ├── PowerApps/         # Capturas de la app y de la base de datos
    └── n8n/
        ├── Program1/      # Tipo de cambio: workflows, notebook, medida DAX
        └── Program2/      # Analista con Claude: workflows, notebook nb_kpis
```

**Licencia de los datos:** el dataset es de Olist y se publica en Kaggle bajo licencia CC BY-NC-SA 4.0.
