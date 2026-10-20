# Enterprise Data Automation

**Excel · Microsoft Fabric · Power BI · SQL · Power Automate · Power Apps · n8n · Claude**

A business intelligence system built on the [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce): about 100,000 real orders from 2016 to 2018, spread across 9 related tables.

The business question that drives the whole project:

> **Why do some customers rate their orders poorly, and which sellers and categories cause the most delays?**

The project moves through four parts. First, the problem is **explored** in Excel. Then it is **measured** with a Power BI report on Fabric. Next, it is **acted on** through an app where each seller handles their bad reviews. Finally, it is **explained** by an automated analyst that uses Claude to answer questions about the data.

---

## Contents

1. [Exploration in Excel](#1-exploration-in-excel)
2. [Fabric Lakehouse and Power BI report](#2-fabric-lakehouse-and-power-bi-report)
3. [Seller portal: chat with the customer](#3-seller-portal-chat-with-the-customer)
4. [AI analyst: n8n + Claude](#4-ai-analyst-n8n--claude)
5. [Findings](#5-findings)
6. [Problems solved along the way](#6-problems-solved-along-the-way)
7. [Repository structure](#7-repository-structure)

---

## Tools

| Tool | Role in the project |
|---|---|
| **Excel** | Exploration and dashboard prototype (Power Query, Power Pivot, pivot tables) |
| **Fabric** | Lakehouse with bronze → silver → gold layers, Dataflows Gen2, notebooks and pipelines |
| **Power BI** | Star schema in Direct Lake mode, DAX measures and executive report |
| **SQL database (Fabric)** | Complaint-tracking tables that the app and the flows can write to |
| **Power Automate** | Seller notifications and email conversations with customers |
| **Power Apps** | Portal where sellers reply to and chat with customers |
| **n8n** | Daily exchange rate and orchestration of the AI analyst |
| **Claude (API)** | Analyst that reads the model's KPIs and writes the diagnosis |

---

## 1. Exploration in Excel

**Goal:** understand the data and prototype the dashboard before building the cloud solution.

- **Power Query:** cleaning of the 9 tables, with fixed date types, duplicates removed and a check for inconsistent dates (for example, a delivery dated before the purchase). Calculated columns were added: `delivery_days`, `delay_days` and `is_late_delivery`.
- **Power Pivot:** related data model and basic measures (sales, orders, average rating).
- **Prototype dashboard:** KPIs, date and state slicers, and one chart per business question.

<p align="center"><img src="Appx/Excel/imgs/Dashbord%20Excel%20Prototype.png" alt="Excel prototype" width="801"></p>

<details>
<summary>More Power Query screenshots</summary>

<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Schema.png" alt="Power Query schema" width="304"></p>
<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2001.png" alt="Power Query sequence 1" width="507"></p>
<p align="center"><img src="Appx/Excel/imgs/Excel%20Power%20Query%20Sequence%2002.png" alt="Power Query sequence 2" width="359"></p>
<p align="center"><img src="Appx/Excel/imgs/Other%20dynamic%20Table.png" alt="Pivot table" width="386"></p>

</details>

📄 M code: [`Appx/Excel/excel_power_query.txt`](Appx/Excel/excel_power_query.txt) · 📊 Workbook: [`analysis/Analysis E-commerce Dashboard Prototype.xlsx`](analysis/) · 🐍 Python EDA: [`analysis/data_eda.ipynb`](analysis/data_eda.ipynb)

---

## 2. Fabric Lakehouse and Power BI report

**Goal:** move the prototype to a medallion architecture in Microsoft Fabric and publish the executive report.

**Lakehouse (medallion)**
- **Bronze:** the 9 CSV files exactly as they come from Kaggle, in `Files/bronze/`.
- **Silver:** cleaning with **Dataflows Gen2**, reusing the same Power Query logic from the Excel prototype.
- **Gold:** Delta tables ready for the model, including a `calendar` table.

<p align="center"><img src="Appx/Fabric/imgs/Lakehouse%20Raw%20Tables%20(Bronze).png" alt="Bronze tables" width="660"></p>
<p align="center"><img src="Appx/Fabric/imgs/Lakehouse%20Processed%20Tables%20(Gold).png" alt="Gold tables" width="213"></p>

**Semantic model**
- **Mode:** Direct Lake on top of the gold layer.
- **Relationships:** many-to-one with single filter direction. Measures that need to filter "upward" use `CROSSFILTER` only inside their own calculation.
- **DAX measures:** sales, average ticket, unique customers (using `customer_unique_id`), % late deliveries, delivery days, average rating and, in part 4, **Sales USD**.

<p align="center"><img src="Appx/Fabric/imgs/Semantic%20Model%20Schema.png" alt="Semantic model" width="409"></p>

**Executive report**

<p align="center"><img src="Appx/Fabric/imgs/Dashboard%20Power%20BI.png" alt="Power BI report" width="798"></p>

<details>
<summary>More Dataflow screenshots</summary>

<p align="center"><img src="Appx/Fabric/imgs/DataFlow%20Execution%20(Silver).png" alt="Dataflow run (silver)" width="667"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Schema.png" alt="Power Query schema in Fabric" width="430"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2001.png" alt="Sequence 1" width="810"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2002.png" alt="Sequence 2" width="735"></p>
<p align="center"><img src="Appx/Fabric/imgs/Fabric%20Power%20Query%20Sequence%2003.png" alt="Sequence 3" width="674"></p>

</details>

📄 Measures and relationships: [`Appx/Fabric/measures_dax.txt`](Appx/Fabric/measures_dax.txt) · 📄 Dataflow: [`Appx/Fabric/dataflow_fabric_power_query.txt`](Appx/Fabric/dataflow_fabric_power_query.txt) · 📑 Report: [`Appx/Fabric/Reporte.pdf`](Appx/Fabric/Reporte.pdf)

---

## 3. Seller portal: chat with the customer

**Goal:** go from *measuring* bad reviews to *handling* them. Whenever a seller gets a 1- or 2-star review, they receive an automatic notice and can contact the customer from an app.

The flow works like this:

1. Power Automate finds 1–2★ reviews that haven't been notified and emails each seller their complaints with a link to the app.
2. The seller replies from the app's chat. The message is emailed to the customer and saved in the database.
3. The customer answers the email (`[Queja #123]` in the subject), and Power Automate saves the reply to the chat.
4. The seller closes the complaint once it is resolved.

### 3.1 Tracking tables (SQL database in Fabric)

The Lakehouse SQL endpoint is **read-only**, so the app and the flows could not write to it. That's why a **Fabric SQL database** was created with three tables:

| Table | Contents |
|---|---|
| `vendedor_contacto` | Demo sellers and their email |
| `seguimiento_resenas` | One row per 1–2★ review and seller, with status, reply and dates |
| `interacciones` | Chat messages between seller and customer |

A complaint moves through these states: `sin_respuesta` (no reply) → `en_contacto` (in contact) → `cerrada` (closed).

Data is loaded with a **Copy job** from the Lakehouse into staging tables, then filtered and joined in T-SQL. There are also **stored procedures**, so Power Automate doesn't have to build queries.

<p align="center"><img src="Appx/Fabric/imgs/copia_tablas_lakehouse_a_sql.png" alt="Lakehouse to SQL copy" width="558"></p>
<p align="center"><img src="Appx/Fabric/imgs/tablas_y_procedimientos_seguimientos.png" alt="Tables and procedures" width="160"></p>

> **Simulated data:** Olist does not include seller or customer emails. The demo uses the 5 sellers with the most bad reviews (1,376 complaints in total) and `+` aliases of a single email account, such as `account+vendedor1@gmail.com`.

📄 [`tablas_seguimiento_sql.txt`](Appx/Fabric/tablas_seguimiento_sql.txt) · 📄 [`procedimientos_tablas_seguimiento_sql.txt`](Appx/Fabric/procedimientos_tablas_seguimiento_sql.txt)

### 3.2 Notices and emails (Power Automate)

| Flow | Trigger | What it does |
|---|---|---|
| **Seller notice** | Scheduled | Sends one email per seller with their pending complaints and the app link, then marks them as notified |
| **Customer reply** | Incoming email with `[Queja #` in the subject | Extracts the complaint number from the subject and saves the reply in `interacciones` |

<p align="center"><img src="Appx/Automate/imgs/lista_flujos_automate.png" alt="Power Automate flows" width="434"></p>
<p align="center"><img src="Appx/Automate/imgs/avisos_a_vendedores.png" alt="Seller notice" width="302"></p>
<p align="center"><img src="Appx/Automate/imgs/respuestas_del_cliente.png" alt="Customer replies" width="495"></p>

<details>
<summary>More screenshots</summary>

<p align="center"><img src="Appx/Automate/imgs/db_seguimiento_a_correo_vendedor.png" alt="From the database to the seller email" width="430"></p>
<p align="center"><img src="Appx/Automate/imgs/correo_a_db_interacciones.png" alt="From the email to the interactions table" width="420"></p>

</details>

📄 Flow expressions and solved errors: [`Appx/Automate/fx_expressions_used_in_flows.txt`](Appx/Automate/fx_expressions_used_in_flows.txt)

### 3.3 Seller app (Power Apps)

Canvas app with three screens:
- **List:** the seller's complaints, filtered by status.
- **Detail:** the full complaint and the button to reply to the customer.
- **Chat:** the email conversation shown as a chat, plus the button to close the complaint.

<p align="center"><img src="Appx/PowerApps/lista_sin_respuesta.png" alt="Complaint list" width="786"></p>
<p align="center"><img src="Appx/PowerApps/detalle_responder.png" alt="Reply screen" width="786"></p>
<p align="center"><img src="Appx/PowerApps/chat_respuesta_cliente.png" alt="Chat with the customer's reply" width="786"></p>

**The full cycle, from three sides:** the customer's inbox, the mailbox Power Automate reads, and the database.

<p align="center"><img src="Appx/PowerApps/gmail_conversacion_cliente.png" alt="Customer inbox (Gmail)" width="798"></p>
<p align="center"><img src="Appx/PowerApps/outlook_respuesta_cliente.png" alt="Mailbox read by Power Automate (Outlook)" width="798"></p>
<p align="center"><img src="Appx/PowerApps/sql_conversacion_queja.png" alt="Conversation in SQL" width="645"></p>

<details>
<summary>More screenshots</summary>

<p align="center"><img src="Appx/PowerApps/chat_mensaje_vendedor.png" alt="Seller's first message" width="786"></p>
<p align="center"><img src="Appx/PowerApps/lista_en_contacto.png" alt="In-contact list" width="786"></p>
<p align="center"><img src="Appx/PowerApps/lista_cerrada.png" alt="Closed list" width="798"></p>
<p align="center"><img src="Appx/PowerApps/sql_resumen_por_vendedor.png" alt="Summary per seller" width="645"></p>
<p align="center"><img src="Appx/PowerApps/sql_quejas_atendidas.png" alt="Handled complaints" width="645"></p>

</details>

---

## 4. AI analyst: n8n + Claude

**Goal:** let anyone ask the report questions in natural language and get a diagnosis grounded in the model's data.

### 4.1 Program 1: daily exchange rate

Olist sales are in Brazilian reais (BRL). To show them in dollars, n8n fetches the official **PTAX** rate from the Banco Central do Brasil.

```
Central Bank API (11:00) → n8n → Google Sheets → notebook (12:00) → Lakehouse → Direct Lake model
```

- **Historical load:** run once; it fetched 750 daily rates from 2016 to 2018.
- **Daily load:** runs Monday to Friday and adds the day's rate.
- **Why a notebook:** a Direct Lake model can only read OneLake tables, so the `nb_tipo_cambio` notebook copies the sheet into the Lakehouse.
- **`Ventas USD` measure:** divides each day's sales by that day's rate. On weekends and holidays it uses the last business day's rate.

<p align="center"><img src="Appx/n8n/Program1/imgs/n8n_carga_historica.png" alt="n8n historical load" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/n8n_diario.png" alt="n8n daily load" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/dax_ventas_usd.png" alt="Sales in BRL and USD by year" width="798"></p>

<details>
<summary>More screenshots</summary>

<p align="center"><img src="Appx/n8n/Program1/imgs/sheets_tipo_cambio.png" alt="Google Sheets" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/fabric_notebook.png" alt="Notebook" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/notebook_programacion.png" alt="Notebook schedule" width="798"></p>
<p align="center"><img src="Appx/n8n/Program1/imgs/modelo_con_tipo_cambio.png" alt="Model with tipo_cambio" width="798"></p>

</details>

📄 [`MedidaDAX.txt`](Appx/n8n/Program1/MedidaDAX.txt) · ⚙️ [`Tipo_cambio_carga_historica.json`](Appx/n8n/Program1/Tipo_cambio_carga_historica.json) · ⚙️ [`Tipo_cambio_diario.json`](Appx/n8n/Program1/Tipo_cambio_diario.json) · 📓 [`nb_tipo_cambio.html`](Appx/n8n/Program1/nb_tipo_cambio.html)

### 4.2 Program 2: Claude analyst

**The challenge.** The original idea was for n8n to query Power BI through its REST API, but that requires registering an app in Microsoft Entra, and the university tenant blocks it.

**The solution.** The model is read by a **Fabric notebook**, which runs with the user's own session and needs no extra permissions. It uses **Semantic Link** (`sempy`) to run DAX against the semantic model and **pushes** the data to n8n through a webhook protected by a header key.

```
pl_diario (12:00, Mon–Fri) → nb_tipo_cambio → nb_kpis (DAX via Semantic Link)
  → n8n webhook (POST + X-Api-Key) → cubo_olist (Google Sheets)

Form → n8n query workflow → Claude → on-screen answer + email + link to the filtered report
```

**1. The KPI cube.** The notebook doesn't send pre-computed percentages but their **components** (sales, orders, late deliveries, total deliveries, rating sum, reviews…), in two tables: *state × month* and *category × month*. That way n8n can recompute any KPI for any filter combination without errors. Averaging pre-computed percentages would give wrong results.

Notebook code and output: [`nb_kpis.html`](Appx/n8n/Program2/nb_kpis.html)

**2. Always up to date.** The `pl_diario` pipeline runs `nb_tipo_cambio` and then `nb_kpis`, so the cube is never older than the model. Every answer includes the data cut-off date.

<p align="center"><img src="Appx/n8n/Program2/imgs/fabric_pl_diario_programacion.png" alt="Pipeline schedule" width="798"></p>
<p align="center"><img src="Appx/n8n/Program2/imgs/fabric_pl_diario_ejecucion.png" alt="Pipeline run" width="798"></p>
<p align="center"><img src="Appx/n8n/Program2/imgs/n8n_recepcion_cubo.png" alt="Cube intake in n8n" width="798"></p>

**3. The query.** The user fills in a form: state, year, category, KPIs, analysis type (executive summary, year-over-year comparison or problem detection), a free-text question and their email. n8n filters the cube, recomputes the selected KPIs and builds the context for Claude, along with a link to the Power BI report **already filtered**.

<p align="center"><img src="Appx/n8n/Program2/imgs/n8n_analista_consulta.png" alt="Query workflow" width="858"></p>

> The model node shows as deactivated in the screenshot and in the JSON because the API key was removed before exporting.

**4. The answer.** Claude is instructed to use **only** the data in the JSON and not to invent causes or figures. In testing, the question was *"Why did the rating drop in São Paulo?"*. Claude **pushed back on the premise**: SP's yearly rating actually rose from 4.15 to 4.19. It then pointed out the real dip in February and March 2018 and tied it to the rise in late deliveries.

<p align="center"><img src="Appx/n8n/Program2/imgs/formulario_analista.png" alt="Form" width="168"></p>
<p align="center"><img src="Appx/n8n/Program2/imgs/correo_analisis_claude.png" alt="Email with the analysis" width="708"></p>

⚙️ [`Analista_recepcion_cubo.json`](Appx/n8n/Program2/Analista_recepcion_cubo.json) · ⚙️ [`Analista_consulta.json`](Appx/n8n/Program2/Analista_consulta.json) · 📓 [`nb_kpis.html`](Appx/n8n/Program2/nb_kpis.html)

---

## 5. Findings

1. **Delays destroy satisfaction.** On-time orders average about **4.3 stars**; orders 6 to 15 days late drop to around **1.8**.
2. **Punctuality got worse as the business grew.** Sales went from R$ 6.16 M in 2017 to R$ 7.39 M in 2018 (January to August), while late deliveries rose from **6.6% to 9.3%**.
3. **February and March 2018 were the crisis.** Nationwide, late deliveries reached **16% and 21.4%**, and the rating fell to **3.83 and 3.75**, the lowest of the period.
4. **The problem is geographic.** Northern and northeastern states concentrate the delays: Alagoas (**23.2%** in 2018), Maranhão and Ceará are above 20%.
5. **One category stands out as the worst.** `office_furniture` has the lowest rating in 2018, with a **3.61** average and **23.6%** negative reviews.
6. **The exchange rate changes the picture.** Measured in dollars, 2018 growth is smaller, because the real weakened from ~3.20 to ~3.50 per dollar.

---

## 6. Repository structure

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

> Some names in the code and in the app are in Spanish (tables, columns and states such as `seguimiento_resenas` or `sin_respuesta`), because they match the actual objects in Fabric, SQL and n8n.

**Data license:** the dataset belongs to Olist and is published on Kaggle under the CC BY-NC-SA 4.0 license.
