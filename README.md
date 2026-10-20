# Enterprise Data Automation

![Excel](https://img.shields.io/badge/Excel-217346?logo=microsoftexcel&logoColor=white)
![Microsoft Fabric](https://img.shields.io/badge/Microsoft%20Fabric-0078D4?logo=microsoft&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)
![SQL](https://img.shields.io/badge/T--SQL-CC2927?logo=microsoftsqlserver&logoColor=white)
![Power Automate](https://img.shields.io/badge/Power%20Automate-0066FF?logo=powerautomate&logoColor=white)
![Power Apps](https://img.shields.io/badge/Power%20Apps-742774?logo=powerapps&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-EA4B71?logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude%20API-D97757?logo=anthropic&logoColor=white)

An end-to-end business intelligence system built on the [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce): ~100,000 real orders (2016–2018) across 9 related tables.

> **Business question:** Why do some customers rate their orders poorly, and which sellers and categories cause the most delays?

The project answers it in four steps:

| Step | What happens | Tools |
|---|---|---|
| **1. Explore** | Clean the data and prototype the dashboard | Excel, Power Query, Power Pivot, Python |
| **2. Measure** | Medallion lakehouse and executive report | Fabric, Dataflows Gen2, Power BI (Direct Lake), DAX |
| **3. Act** | Sellers get notified of bad reviews and chat with the customer by email | Fabric SQL, Power Automate, Power Apps |
| **4. Explain** | Ask questions in natural language and get a data-grounded diagnosis | n8n, Fabric notebooks, Semantic Link, Claude |

---

## Key findings

- **Delays destroy satisfaction:** on-time orders average **~4.3★**; orders 6–15 days late fall to **~1.8★**.
- **Growth hurt punctuality:** sales rose from R$ 6.16 M (2017) to R$ 7.39 M (Jan–Aug 2018) while late deliveries went from **6.6% to 9.3%**.
- **Feb–Mar 2018 was the crisis:** late deliveries hit **16% and 21.4%** nationwide, and ratings dropped to **3.83 and 3.75**.
- **The problem is geographic:** the North and Northeast concentrate delays (Alagoas **23.2%** in 2018; Maranhão and Ceará above 20%).

[See all findings ↓](#6-findings)

---

## Architecture

```mermaid
flowchart LR
    K[(Kaggle CSVs)] --> B

    subgraph Fabric["Microsoft Fabric"]
        B[Bronze<br/>raw files] -->|Dataflows Gen2| S[Silver<br/>clean]
        S --> G[Gold<br/>Delta tables]
        G --> M[Semantic model<br/>Direct Lake]
        G -->|Copy job| SQL[(SQL database<br/>complaint tracking)]
        NB1[nb_tipo_cambio] --> G
        NB2[nb_kpis<br/>Semantic Link] -.reads.-> M
    end

    M --> PBI[Power BI report]

    SQL <--> PA[Power Automate]
    SQL <--> APP[Power Apps<br/>seller portal]
    PA <-->|email| C((Customer))
    PA -->|notice| V((Seller))
    V --> APP

    BCB[Banco Central do Brasil<br/>PTAX API] --> N1[n8n · exchange rate]
    N1 --> GS1[Google Sheets] --> NB1

    NB2 -->|webhook| N2[n8n · KPI cube]
    F[Form] --> N3[n8n · query] --> CL[Claude] --> R[Answer + email +<br/>filtered report link]
    N2 --> N3
```

---

## Contents

1. [Exploration in Excel](#1-exploration-in-excel)
2. [Fabric Lakehouse and Power BI report](#2-fabric-lakehouse-and-power-bi-report)
3. [Seller portal: chat with the customer](#3-seller-portal-chat-with-the-customer)
4. [AI analyst: n8n + Claude](#4-ai-analyst-n8n--claude)
5. [Challenges and how they were solved](#5-challenges-and-how-they-were-solved)
6. [Findings](#6-findings)
7. [Repository structure](#7-repository-structure)
8. [Reproducing the project](#8-reproducing-the-project)

---

## 1. Exploration in Excel

**Goal:** understand the data and prototype the dashboard before building the cloud solution.

- **Power Query:** cleaning of the 9 tables — date types fixed, duplicates removed, and inconsistent dates flagged (e.g. a delivery dated before the purchase). Calculated columns: `delivery_days`, `delay_days`, `is_late_delivery`.
- **Prototype dashboard:** KPIs, date and state slicers, one chart per business question.

<p align="center"><img src="Appx/Excel/imgs/Dashbord%20Excel%20Prototype.png" alt="Excel prototype dashboard" width="801"></p>

<details>
<summary>Power Query & Dynamic Tables screenshots</summary>

<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Schema.png" alt="Power Query schema" width="304"></p>
<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2001.png" alt="Power Query sequence 1" width="507"></p>
<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2002.png" alt="Power Query sequence 2" width="359"></p>
<p align="center"><img src="Appx/Excel/imgs/Other%20dynamic%20Table.png" alt="Pivot table" width="386"></p>

</details>

**Files:** [M code](Appx/Excel/excel_power_query.txt) · [Excel workbook](analysis/) · [Python EDA notebook](analysis/data_eda.ipynb)

---

## 2. Fabric Lakehouse and Power BI report

**Goal:** move the prototype to a medallion architecture in Microsoft Fabric and publish the executive report.

### Lakehouse (medallion)

| Layer | Content |
|---|---|
| **Bronze** | The 9 Kaggle CSVs as-is, in `Files/bronze/` |
| **Silver** | Cleaned with **Dataflows Gen2**, reusing the Power Query logic from the Excel prototype |
| **Gold** | Model-ready Delta tables, including a `calendar` table |

<p align="center"><img src="Appx/Fabric/imgs/Lakehouse%20Raw%20Tables%20(Bronze).png" alt="Bronze tables" width="660"></p>
<p align="center"><img src="Appx/Fabric/imgs/Lakehouse%20Processed%20Tables%20(Gold).png" alt="Gold tables" width="213"></p>

### Semantic model

- **Direct Lake** mode on top of the gold layer.
- **Many-to-one relationships with single filter direction.** Measures that need to filter "upward" enable `CROSSFILTER` only inside their own calculation, keeping the model predictable.
- **DAX measures:** sales, average ticket, unique customers (via `customer_unique_id`), % late deliveries, delivery days, average rating and — added in part 4 — **Sales USD**.

<p align="center"><img src="Appx/Fabric/imgs/Semantic%20Model%20Schema.png" alt="Semantic model" width="409"></p>

### Executive report

<p align="center"><img src="Appx/Fabric/imgs/Dashboard%20Power%20BI.png" alt="Power BI report" width="798"></p>

<details>
<summary>Dataflow screenshots</summary>

<p align="center"><img src="Appx/Fabric/imgs/DataFlow%20Execution%20(Silver).png" alt="Dataflow run (silver)" width="667"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Schema.png" alt="Power Query schema in Fabric" width="430"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2001.png" alt="Sequence 1" width="810"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2002.png" alt="Sequence 2" width="735"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2003.png" alt="Sequence 3" width="674"></p>

</details>

**Files:** [DAX measures and relationships](Appx/Fabric/measures_dax.txt) · [Dataflow M code](Appx/Fabric/dataflow_fabric_power_query.txt) · [Report (PDF)](Appx/Fabric/Reporte.pdf)

---

## 3. Seller portal: chat with the customer

**Goal:** go from *measuring* bad reviews to *handling* them. When a seller gets a 1★ or 2★ review, they are notified automatically and can contact the customer from an app.

```mermaid
sequenceDiagram
    participant DB as Fabric SQL
    participant PA as Power Automate
    participant S as Seller (Power Apps)
    participant C as Customer (email)

    PA->>DB: sp_avisos_pendientes (every 15 min)
    PA->>S: Email with pending complaints + app link
    PA->>DB: sp_marcar_notificadas
    S->>DB: Reply saved (status: en_contacto)
    S->>C: Email "[Queja #123] ..."
    C->>PA: Reply to the email
    PA->>DB: sp_guardar_respuesta_cliente
    DB-->>S: Message appears in the chat
    S->>DB: Close complaint (status: cerrada)
```

### 3.1 Tracking tables (SQL database in Fabric)

| Table | Contents |
|---|---|
| `vendedor_contacto` | Demo sellers and their email |
| `seguimiento_resenas` | One row per 1–2★ review and seller: status, reply, dates |
| `interacciones` | Chat messages between seller and customer |

Complaint lifecycle: `sin_respuesta` (no reply) → `en_contacto` (in contact) → `cerrada` (closed).

Data arrives through a **Copy job** from the Lakehouse into staging tables, then is filtered and joined in T-SQL. **Stored procedures** encapsulate every read and write, so the flows never build queries themselves.

<p align="center"><img src="Appx/Fabric/imgs/copia_tablas_lakehouse_a_sql.png" alt="Lakehouse to SQL copy" width="558"></p>
<p align="center"><img src="Appx/Fabric/imgs/tablas_y_procedimientos_seguimientos.png" alt="Tables and procedures" width="160"></p>

> **Simulated contacts:** Olist does not include seller or customer emails. The demo uses the 5 sellers with the most bad reviews (1,376 complaints in total) and `+` aliases of a single mailbox, such as `account+vendedor1@gmail.com`.

**Files:** [Table definitions](Appx/Fabric/tablas_seguimiento_sql.txt) · [Stored procedures](Appx/Fabric/procedimientos_tablas_seguimiento_sql.txt)

### 3.2 Notifications and emails (Power Automate)

| Flow | Trigger | What it does |
|---|---|---|
| **Seller notice** | Scheduled, every 15 minutes | One email per seller with their pending complaints and the app link; marks them as notified |
| **Customer reply** | Incoming email with `[Queja #` in the subject | Extracts the complaint number from the subject and saves the reply in `interacciones` |

<p align="center"><img src="Appx/Automate/imgs/lista_flujos_automate.png" alt="Power Automate flows" width="434"></p>

<details>
<summary>Flow screenshots</summary>

<p align="center"><img src="Appx/Automate/imgs/avisos_a_vendedores.png" alt="Seller notice flow" width="302"></p>
<p align="center"><img src="Appx/Automate/imgs/respuestas_del_cliente.png" alt="Customer reply flow" width="495"></p>
<p align="center"><img src="Appx/Automate/imgs/db_seguimiento_a_correo_vendedor.png" alt="From the database to the seller email" width="430"></p>
<p align="center"><img src="Appx/Automate/imgs/correo_a_db_interacciones.png" alt="From the email to the interactions table" width="420"></p>

</details>

**Files:** [Flow expressions](Appx/Automate/fx_expressions_used_in_flows.txt)

### 3.3 Seller app (Power Apps)

Canvas app with three screens:

| Screen | Purpose |
|---|---|
| **List** | The seller's complaints, filtered by status |
| **Detail** | The full complaint and the button to reply |
| **Chat** | The email conversation rendered as a chat, plus the button to close the complaint |

<p align="center"><img src="Appx/PowerApps/lista_sin_respuesta.png" alt="Complaint list" width="786"></p>
<p align="center"><img src="Appx/PowerApps/detalle_responder.png" alt="Reply screen" width="786"></p>
<p align="center"><img src="Appx/PowerApps/chat_respuesta_cliente.png" alt="Chat with the customer's reply" width="786"></p>

<details>
<summary>The full cycle from three sides: customer inbox, monitored mailbox and database</summary>

<p align="center"><img src="Appx/PowerApps/gmail_conversacion_cliente.png" alt="Customer inbox (Gmail)" width="798"></p>
<p align="center"><img src="Appx/PowerApps/outlook_respuesta_cliente.png" alt="Mailbox read by Power Automate (Outlook)" width="798"></p>
<p align="center"><img src="Appx/PowerApps/sql_conversacion_queja.png" alt="Conversation in SQL" width="645"></p>

</details>

<details>
<summary>More app and database screenshots</summary>

<p align="center"><img src="Appx/PowerApps/chat_mensaje_vendedor.png" alt="Seller's first message" width="786"></p>
<p align="center"><img src="Appx/PowerApps/lista_en_contacto.png" alt="In-contact list" width="786"></p>
<p align="center"><img src="Appx/PowerApps/lista_cerrada.png" alt="Closed list" width="798"></p>
<p align="center"><img src="Appx/PowerApps/sql_resumen_por_vendedor.png" alt="Summary per seller" width="645"></p>
<p align="center"><img src="Appx/PowerApps/sql_quejas_atendidas.png" alt="Handled complaints" width="645"></p>

</details>

---

## 4. AI analyst: n8n + Claude

**Goal:** let anyone ask questions about the data in natural language and get a diagnosis grounded in the model's numbers.

### 4.1 Program 1 — Daily exchange rate

Olist sales are in Brazilian reais (BRL). To report them in dollars, n8n fetches the official **PTAX** rate from the Banco Central do Brasil.

```
BCB API (11:00) → n8n → Google Sheets → nb_tipo_cambio (12:00) → Lakehouse → Direct Lake model
```

- **Historical load:** run once; loaded 750 daily rates for 2016–2018.
- **Daily load:** Monday to Friday, appends the day's rate.
- **`Ventas USD` measure:** divides each day's sales by that day's rate; weekends and holidays use the last business day's rate.

<p align="center"><img src="Appx/n8n/Program1/imgs/dax_ventas_usd.png" alt="Sales in BRL and USD by year" width="798"></p>

<details>
<summary>Workflow, notebook and model screenshots</summary>

<p align="center"><img src="Appx/n8n/Program1/imgs/n8n_carga_historica.png" alt="n8n historical load" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/n8n_diario.png" alt="n8n daily load" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/sheets_tipo_cambio.png" alt="Google Sheets" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/fabric_notebook.png" alt="Notebook" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/notebook_programacion.png" alt="Notebook schedule" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/modelo_con_tipo_cambio.png" alt="Model with tipo_cambio" width="798"></p>

</details>

**Files:** [DAX measure](Appx/n8n/Program1/MedidaDAX.txt) · [Historical workflow](Appx/n8n/Program1/Tipo_cambio_carga_historica.json) · [Daily workflow](Appx/n8n/Program1/Tipo_cambio_diario.json) · [Notebook](Appx/n8n/Program1/nb_tipo_cambio.html)

### 4.2 Program 2 — Claude analyst

```
pl_diario (12:00, Mon–Fri) → nb_tipo_cambio → nb_kpis (DAX via Semantic Link)
  → n8n webhook (POST + X-Api-Key) → cubo_olist (Google Sheets)

Form → n8n query workflow → Claude → on-screen answer + email + link to the filtered report
```

**1. A KPI cube built from components.** The `nb_kpis` notebook does not send pre-computed percentages; it sends their **components** (sales, orders, late deliveries, total deliveries, rating sum, review count…) in two tables: *state × month* and *category × month*. n8n can then recompute any KPI for any combination of filters correctly — averaging pre-computed percentages would not.

**2. Always in sync with the model.** The `pl_diario` pipeline runs `nb_tipo_cambio` and then `nb_kpis`, so the cube is never older than the model. Every answer states the data cut-off date.

<p align="center"><img src="Appx/n8n/Program2/imgs/fabric_pl_diario_ejecucion.png" alt="Pipeline run" width="798"></p>

**3. The query.** The user picks state, year, category, KPIs and analysis type (executive summary, year-over-year comparison or problem detection), writes a free-text question and leaves an email. n8n filters the cube, recomputes the KPIs, builds the context for Claude and generates a link to the Power BI report **already filtered**.

<p align="center"><img src="Appx/n8n/Program2/imgs/n8n_analista_consulta.png" alt="Query workflow" width="858"></p>

> The model node appears deactivated in the screenshot and JSON because the API key was removed before exporting.

**4. The answer.** Claude is instructed to use **only** the data in the JSON and never invent causes or figures. Asked *"Why did the rating drop in São Paulo?"*, Claude **challenged the premise**: SP's yearly rating actually rose from 4.15 to 4.19. It then identified the real dip in February–March 2018 and tied it to the spike in late deliveries.

<p align="center">
  <img src="Appx/n8n/Program2/imgs/formulario_analista.png" alt="Form" width="168">
  &nbsp;&nbsp;
  <img src="Appx/n8n/Program2/imgs/correo_analisis_claude.png" alt="Email with the analysis" width="708">
</p>

<details>
<summary>Pipeline and cube intake screenshots</summary>

<p align="center"><img src="Appx/n8n/Program2/imgs/fabric_pl_diario_programacion.png" alt="Pipeline schedule" width="798"></p>
<p align="center"><img src="Appx/n8n/Program2/imgs/n8n_recepcion_cubo.png" alt="Cube intake in n8n" width="798"></p>
<p align="center"><img src="Appx/n8n/Program2/imgs/n8n_ejecucion_produccion.png" alt="Production run in n8n" width="798"></p>

</details>

**Files:** [Cube intake workflow](Appx/n8n/Program2/Analista_recepcion_cubo.json) · [Query workflow](Appx/n8n/Program2/Analista_consulta.json) · [nb_kpis notebook](Appx/n8n/Program2/nb_kpis.html)

---

## 5. Challenges and how they were solved

| Challenge | Solution |
|---|---|
| The Lakehouse SQL endpoint is **read-only**, so the app and flows couldn't write complaint status or messages. | Created a separate **Fabric SQL database**, loaded via Copy job, with stored procedures for every read/write. |
| Olist has **no seller or customer emails**. | Simulated contacts with `+` aliases of a single mailbox for the 5 sellers with the most complaints. |
| Linking a customer's email reply to the right complaint. | The complaint ID travels in the subject (`[Queja #123]`); the flow extracts it with an expression and stores the reply. |
| A **Direct Lake** model can only read OneLake tables, not Google Sheets. | The `nb_tipo_cambio` notebook copies the exchange-rate sheet into the Lakehouse daily. |
| No exchange rate on **weekends and holidays**. | The `Ventas USD` measure falls back to the last business day's rate. |
| The Power BI REST API needs an **Entra app registration**, blocked by the university tenant. | Inverted the direction: a Fabric notebook reads the model with **Semantic Link** under the user's session and **pushes** the data to an n8n webhook protected by `X-Api-Key`. |
| Averaging pre-computed percentages gives **wrong KPIs** when filters are combined. | The cube stores numerators and denominators; n8n recomputes each ratio after filtering. |
| Risk of the LLM **inventing causes or numbers**. | Claude receives only the filtered JSON, with explicit instructions to stick to it; every answer includes the data cut-off date and a link to the filtered report to verify. |

---

## 6. Findings

1. **Delays destroy satisfaction.** On-time orders average about **4.3★**; orders 6 to 15 days late drop to around **1.8★**.
2. **Punctuality got worse as the business grew.** Sales went from R$ 6.16 M in 2017 to R$ 7.39 M in 2018 (January to August), while late deliveries rose from **6.6% to 9.3%**.
3. **February and March 2018 were the crisis.** Nationwide, late deliveries reached **16% and 21.4%**, and the rating fell to **3.83 and 3.75**, the lowest of the period.
4. **The problem is geographic.** Northern and northeastern states concentrate the delays: Alagoas (**23.2%** in 2018), Maranhão and Ceará are above 20%.
5. **One category stands out as the worst.** `office_furniture` has the lowest rating in 2018, with a **3.61** average and **23.6%** negative reviews.
6. **The exchange rate changes the picture.** Measured in dollars, 2018 growth is smaller, because the real weakened from ~3.20 to ~3.50 per dollar.

---

## 7. Repository structure

```
├── kaggle_data/           # 9 original Olist CSV files
├── analysis/              # Excel prototype, Python EDA and PDF report
└── Appx/
    ├── Excel/             # Prototype Power Query and screenshots
    ├── Fabric/            # Dataflow, DAX measures, tracking SQL and screenshots
    ├── Automate/          # Flow expressions and screenshots
    ├── PowerApps/         # App and database screenshots
    └── n8n/
        ├── Program1/      # Exchange rate: workflows, notebook, DAX measure
        └── Program2/      # Claude analyst: workflows, nb_kpis notebook
```

> Some object names are in Spanish (`seguimiento_resenas`, `sin_respuesta`, `tipo_cambio`…) because they match the actual tables, columns and workflows in Fabric, SQL and n8n.

---

## 8. Reproducing the project

The cloud components depend on licensed services, so the repository contains the code, exports and screenshots needed to rebuild each part rather than a one-click deployment.

| Part | What you need | How |
|---|---|---|
| Exploration | Excel (Microsoft 365) or Python | Open the workbook in `analysis/` or run `data_eda.ipynb` against `kaggle_data/` |
| Lakehouse + report | Microsoft Fabric workspace | Upload the CSVs to `Files/bronze/`, recreate the Dataflow from its M code, then build the model with `measures_dax.txt` |
| Seller portal | Fabric SQL database, Power Automate, Power Apps | Run the table and procedure scripts, then recreate the two flows using the expressions file |
| AI analyst | n8n, Google Sheets, an Anthropic API key | Import the JSON workflows, add your own credentials and API key, and schedule the notebooks in Fabric |

---

**Data license:** the dataset belongs to Olist and is published on Kaggle under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
