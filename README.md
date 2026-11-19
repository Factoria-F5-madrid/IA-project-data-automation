# 📊 Proyecto - Automatización MySQL → Python → Dashboard

## 📝 Descripción del proyecto

Este proyecto automatiza la extracción y el análisis de datos de una base SQL con Python.
Un único comando genera varios archivos CSV y un **dashboard interactivo hecho también en Python**
(Streamlit) que lee esos CSV. No hay pasos manuales en ninguna herramienta externa: si cambian los
datos, se vuelve a ejecutar y todo se actualiza.

**Flujo:** MySQL → ETL en Python → CSV → Dashboard en Streamlit

## 🎯 Objetivos concretos

- Automatizar la extracción de datos desde MySQL
- Generar 3 o más datasets en formato CSV ya procesados
- Construir un dashboard interactivo que lea esos CSV
- Documentar y estructurar el proyecto para que cualquiera pueda ejecutarlo

## 🗄️ El dataset: Olist

Usaremos el dataset público de **Olist**, con datos reales de un marketplace brasileño donde
vendedores pequeños publican productos y los clientes los compran. Son **99.441 pedidos entre 2016 y 2018**.

| Tabla (en MySQL) | Filas | Qué contiene | Una fila representa |
| :--- | ---: | :--- | :--- |
| `customers` | 99.441 | Ciudad y estado del cliente | Un cliente **por pedido** (`customer_id`). La persona real es `customer_unique_id` |
| `orders` | 99.441 | Estado y fechas del pedido (compra, aprobación, envío, entrega real y estimada) | Un pedido |
| `order_items` | 112.650 | Productos de cada pedido, con precio y flete | Un artículo dentro de un pedido |
| `order_payments` | 103.886 | Cómo se pagó (tipo, cuotas, valor) | Un pago (un pedido puede tener varios) |
| `order_reviews` | 99.224 | Puntuación y comentario del cliente | Una reseña |
| `products` | 32.951 | Categoría y medidas del producto | Un producto |
| `sellers` | 3.095 | Ciudad y estado del vendedor | Un vendedor |
| `geolocation` | 1.000.163 | Coordenadas por código postal | Un punto geográfico (hay muchos por código postal) |
| `categoria_traduccion` | 71 | Nombre de categoría en portugués e inglés | Una categoría |

### Instalación

1. Descarguen el dataset de Kaggle: [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
   (necesitan una cuenta gratuita de Kaggle). Descomprimen el `.zip` en una carpeta, por ejemplo `kaggle/`.
2. Creen la base de datos en MySQL:

   ```sql
   CREATE DATABASE olist CHARACTER SET utf8mb4;
   ```

3. Carguen los CSV como tablas. Usen estos nombres para que todos los equipos trabajen igual:

   | Archivo CSV | Tabla en MySQL |
   | :--- | :--- |
   | `olist_customers_dataset.csv` | `customers` |
   | `olist_orders_dataset.csv` | `orders` |
   | `olist_order_items_dataset.csv` | `order_items` |
   | `olist_order_payments_dataset.csv` | `order_payments` |
   | `olist_order_reviews_dataset.csv` | `order_reviews` |
   | `olist_products_dataset.csv` | `products` |
   | `olist_sellers_dataset.csv` | `sellers` |
   | `olist_geolocation_dataset.csv` | `geolocation` |
   | `product_category_name_translation.csv` | `categoria_traduccion` |

   Pueden hacerlo con el asistente de importación de MySQL Workbench o con este script de Python
   (instalen antes `pandas`, `sqlalchemy` y `mysql-connector-python`):

   ```python
   import pandas as pd
   from sqlalchemy import create_engine

   engine = create_engine("mysql+mysqlconnector://root:TU_PASSWORD@localhost/olist")
   tablas = {
       "olist_customers_dataset.csv": "customers",
       "olist_orders_dataset.csv": "orders",
       "olist_order_items_dataset.csv": "order_items",
       "olist_order_payments_dataset.csv": "order_payments",
       "olist_order_reviews_dataset.csv": "order_reviews",
       "olist_products_dataset.csv": "products",
       "olist_sellers_dataset.csv": "sellers",
       "olist_geolocation_dataset.csv": "geolocation",
       "product_category_name_translation.csv": "categoria_traduccion",
   }
   for archivo, tabla in tablas.items():
       df = pd.read_csv(f"kaggle/{archivo}")
       for c in df.columns:
           if c.endswith(("_date", "_timestamp", "_at")):
               df[c] = pd.to_datetime(df[c])  # las fechas llegan como texto
       df.to_sql(tabla, engine, if_exists="replace", index=False, chunksize=10000)
       print(tabla, len(df))
   ```

4. Compruébenlo: `SELECT COUNT(*) FROM orders;` debe dar 99.441.

> [!NOTE]
> Licencia del dataset: CC BY-NC-SA 4.0 (uso no comercial, con atribución a Olist).

## 📊 Análisis objetivo

- Comportamiento de clientes y patrones de compra
- Distribución geográfica de ingresos (estados y ciudades de Brasil)
- Tendencias temporales de pedidos, pagos y tiempos de entrega
- Identificación de vendedores y categorías clave

## 🧭 Antes de programar: definan el grano y los KPIs

### ¿Qué es el grano?

El **grano** (*granularity*) es **qué representa exactamente una fila de una tabla**. Es el nivel de detalle.

Imaginen una compra en el supermercado con 3 productos pagada con 2 tarjetas. Según cómo se registre, hay tablas distintas:

| Si cada fila es… | Filas para esa compra | Grano |
| :--- | ---: | :--- |
| Una compra | 1 | un pedido |
| Un producto del ticket | 3 | un artículo |
| Un pago | 2 | un pago |

Las tres son correctas, pero **no se pueden mezclar sin cuidado**. Si unen artículos con pagos, esa compra pasa a ser 3 × 2 = 6 filas y cada pago se cuenta varias veces.

Regla de oro: antes de escribir una consulta, completen la frase *"cada fila de mi resultado es un/una ______"*.

### Lo que tienen que decidir

El primer trabajo del equipo no es código. Es decidir tres cosas y escribirlas en el README:

1. **El grano de cada dataset** (por ejemplo "un pedido", "un vendedor").
2. **Los KPIs** que quieren mostrar (por ejemplo facturación total, número de pedidos, ticket medio, % de entregas con retraso).
3. **Una consulta SQL por dataset** que respete ese grano.

> [!WARNING]
> **La trampa del grano.** Un pedido puede tener varios artículos, varios pagos y varias reseñas.
> Si hacen `JOIN` de `orders` con `order_items`, `order_payments` y `order_reviews` sin agregar antes,
> cada pedido se repite varias veces y la facturación se infla (en Olist, alrededor de un **26%**).
> En un ETL automatizado ese error se repite en **cada ejecución** y nadie lo revisa.
> Por eso el ETL tiene que **comprobar el grano solo** (ver checklist).

## 🧰 Tecnologías

- **Python**
- **Base de datos**: MySQL
- **Librerías principales**:
  - `pandas` - Manipulación y análisis de datos
  - `sqlalchemy` + `mysql-connector-python` - Conexión a MySQL
  - `python-dotenv` - Variables de entorno
  - `streamlit` - Dashboard interactivo
  - `plotly` - Gráficos
- **Control de versiones**: Git y GitHub

## 👥 Equipos

**Equipos de 3 o 4 personas.**

## 📦 Condiciones de entrega

- Repositorio en GitHub
- Un **dashboard en Streamlit** con:
  - **Mínimo 4 KPIs** arriba (los que definieron en el README)
  - **Mínimo 4 gráficos** que cubran los 4 ejes del análisis (clientes, geografía, tiempo, vendedores/categorías)
  - **Mínimo 2 filtros** interactivos (por ejemplo año y estado)
- **Documentación completa**:
  - README con instrucciones de instalación y uso
  - Explicación del proceso de automatización

## ⏳ Plazo de Entrega

**1 semana**

## 🏗️ Estructura del proyecto

```
proyecto-olist-automation/
│
├── main.py                    ⭐ Ejecuta el ETL (genera los CSV)
│
├── src/                       📦 Código fuente
│   ├── __init__.py
│   ├── olist_ETL.py           (extracción, validación y transformación)
│   └── config.py              (configuración desde .env)
│
├── queries/                   📜 Consultas SQL
│   ├── clientes_actividad.sql
│   ├── catalogo_productos.sql
│   ├── vendedores.sql
│   └── entregas_retrasos.sql
│
├── output/                    📂 CSV generados por el ETL
│
├── dashboard/                 📊 Visualización
│   ├── app.py                 (dashboard Streamlit, lee output/*.csv)
│   └── README.md              (guía del dashboard)
│
├── requirements.txt
├── .env                       🔒 Credenciales (no se sube a Git)
├── .env.example
├── .gitignore
└── README.md
```

**Cómo se ejecuta:**

```bash
python main.py                      # genera los CSV en output/
streamlit run dashboard/app.py      # abre el dashboard
```

**🔒 IMPORTANTE:** `.env` contiene credenciales y NO se sube a Git (va en `.gitignore`).

## ✅ Checklist de desarrollo

### Configuración inicial

- [ ] Instalar y configurar MySQL con la base Olist
- [ ] Crear entorno virtual e instalar dependencias
- [ ] Configurar variables de entorno para la conexión

### Pipeline de datos

- [ ] Implementar la conexión a la base de datos
- [ ] Documentar el grano y los KPIs en el README
- [ ] Una consulta SQL por dataset en `queries/`
- [ ] **Validar el grano en cada ejecución** (por ejemplo: sin duplicados en la clave del grano y facturación total igual a la de pagos)
- [ ] Exportar los CSV a `output/`
- [ ] Script principal `main.py` con manejo de errores y mensajes claros

### Dashboard

- [ ] Cargar los CSV con pandas (con `@st.cache_data`)
- [ ] KPIs principales con `st.metric`
- [ ] Los 4 gráficos con Plotly
- [ ] Los filtros en la barra lateral
- [ ] Documentar cómo ejecutarlo

### 🌟 Extras (opcionales)

- [ ] Una tabla resumen que el usuario pueda ordenar y descargar
- [ ] Un mapa de Brasil con la facturación por estado
- [ ] Un botón para volver a ejecutar el ETL desde el dashboard

## 🧪 Criterios de evaluación

- Configurar y automatizar su entorno de trabajo
- Gestionar equipos técnicos
- Evaluar conjuntos de datos (el grano está definido, se respeta y se valida)
- Desarrollar interfaces dinámicas
