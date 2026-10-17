# Azure Logistics Data Pipeline

🇬🇧 [Read in English](README.md)

Proyecto de ingeniería de datos de punta a punta, desarrollado para la **Diplomatura en Data Engineering (ICARO)**. Simula el flujo de datos de una empresa de logística: ingesta de envíos desde dos fuentes, validación y generación de tablas analíticas listas para reportes de negocio.

> **Estado del proyecto:** finalizado como proyecto académico. Los recursos de Azure se crearon con una suscripción gratuita que ya venció, por lo que el pipeline ya no se puede ejecutar. Este repositorio documenta la implementación: código, arquitectura, decisiones de diseño y resultados.

---

## Caso de negocio

Una empresa que envía paquetes entre ciudades de Argentina necesita entender:

- Cómo se comportan sus envíos (entregados, en tránsito, demorados, cancelados)
- Cómo rinde cada sucursal
- Cómo son las distintas rutas (distancia, duración, costo de combustible, peajes)

## Arquitectura

![Arquitectura](docs/img/arquitectura.png)

```
CSV (envíos) ────┐
                 ├─► Azure Data Factory ─► Landing ─► Bronze ─► Databricks ─► Silver ─► Controles de calidad ─► Gold ─► Visualización
JSON (distancias)┘                                              (PySpark)                                       (Delta)
```

| Capa | Contenido |
|------|-----------|
| **Landing** | Archivos fuente tal como llegan |
| **Bronze** | Datos ingeridos por Data Factory, separados por temática (`envios/`, `distancias/`) |
| **Silver** | Datos limpios, envíos unidos con la información de rutas (Delta) |
| **Gold** | Tablas agregadas orientadas al negocio (Delta) |

## Tecnologías

| Componente | Uso |
|-----------|-----|
| Azure Data Factory | Ingesta y orquestación (actividades Copy Data, Schedule Trigger) |
| Azure Data Lake Storage Gen2 | Almacenamiento de todas las capas |
| Azure Databricks | Procesamiento y transformación |
| PySpark | Limpieza, validación, joins y agregaciones |
| Delta Lake | Formato de Silver y Gold |
| Managed Identity + Unity Catalog | Acceso seguro entre servicios (sin claves del storage en el pipeline) |

## Fuentes de datos

| Fuente | Formato | Campos principales |
|--------|---------|--------------------|
| Envíos | CSV (delimitador `;`) | `id_envio`, `fecha`, `sucursal`, `origen`, `destino`, `peso_kg`, `cliente`, `estado` |
| Rutas ("API") | JSON | `origen`, `destino`, `distancia_km`, `duracion_min`, `peajes`, `costo_combustible_estimado` |

Los datos son **sintéticos** e incluyen registros inválidos a propósito para demostrar los controles de calidad. Los archivos de ejemplo están en [`/data`](data/).

## Orquestación (Azure Data Factory)

![Pipeline ADF](docs/img/pipeline_adf.png)

1. **Copy Envios** y **Copy Distancias** se ejecutan en paralelo y mueven los archivos de Landing a Bronze.
2. **nb_Bronze_Silver** se ejecuta solo si ambas copias terminan correctamente.
3. **nb_Silver_Gold** se ejecuta solo si el notebook anterior termina correctamente.

Un Schedule Trigger (`tr_diario`) ejecuta todo el pipeline una vez por día.

**Seguridad:** Data Factory se autentica contra Databricks y el Data Lake mediante una **Managed Identity**, con permisos `READ FILES` / `WRITE FILES` sobre una External Location de Unity Catalog. En la ejecución automatizada no se usan credenciales personales ni claves del storage.

Las definiciones exportadas del pipeline y del trigger están en [`/adf`](adf/).

## Notebooks

**`nb_bronze_a_silver.ipynb`**
- Lee el CSV de envíos y el JSON de distancias desde Bronze
- Perfilado de datos: conteo de registros, IDs duplicados, nulos por columna, valores distintos de `estado`
- Limpieza: elimina filas con nulos en `fecha`, `origen`, `destino`, `peso_kg` y duplicados por `id_envio`
- Left join de envíos con rutas por `origen` y `destino`
- Escribe el resultado en Silver en formato Delta

**`nb_silver_to_gold.ipynb`**
- Lee Silver
- Controles de calidad mediante columnas auxiliares
- Separa registros válidos e inválidos
- Genera las tres tablas Gold con agregaciones `groupBy` y las guarda en Delta

## Calidad de datos

| Control | Regla |
|---------|-------|
| `valida_id` | `id_envio` no es nulo |
| `valida_peso` | `peso_kg > 0` |
| `valida_estado` | `estado` es uno de: Entregado, En tránsito, Demorado, Cancelado |
| `valida_distancia` | `distancia_km > 0` |

| Registros evaluados | Rechazados | Válidos |
|:-:|:-:|:-:|
| 89 | 2 | **87** |

Los dos registros rechazados (`id_envio` 91 y 45) tenían `peso_kg = -5`. Se excluyen antes de la capa Gold para que no distorsionen las métricas.

## Tablas Gold

| Tabla | Granularidad | Métricas |
|-------|--------------|----------|
| `envios_por_sucursal` | Sucursal | Total de envíos, cantidad por estado, peso y distancia promedio |
| `kpi_envios` | Sucursal | Lo anterior más distancia total, duración promedio, costo de combustible (total y promedio), peajes totales y **porcentaje de entregas** |
| `rutas` | Origen → Destino | Cantidad de envíos, distancia y duración promedio, combustible, peajes, peso promedio |

Ejemplo de salida de `envios_por_sucursal`:

| sucursal | total_envios | entregados | en_transito | demorados | cancelados |
|----------|:-:|:-:|:-:|:-:|:-:|
| Sur | 19 | 2 | 4 | 4 | 9 |
| Oeste | 15 | 0 | 6 | 4 | 5 |
| Este | 18 | 7 | 6 | 4 | 1 |
| Norte | 19 | 5 | 9 | 2 | 3 |
| Centro | 16 | 6 | 2 | 4 | 4 |

## Visualización

Gráfico de barras construido en Databricks sobre `kpi_envios`, con el porcentaje de entregas por sucursal.

![Porcentaje de entregas por sucursal](docs/img/visualizacion_kpi.png)

En este conjunto de datos, Este (38,89 %) y Centro (37,50 %) tienen los mayores porcentajes de entrega, Norte llega al 26,32 %, Sur al 10,53 % y Oeste no registra envíos entregados.

## Estructura del repositorio

```
azure-logistics-data-pipeline/
├── README.md
├── README.es.md
├── notebooks/
│   ├── nb_bronze_a_silver.ipynb
│   └── nb_silver_to_gold.ipynb
├── data/
│   ├── envios_logistica.csv
│   └── api_distancias.json
├── adf/
│   ├── pl_ingesta_bronze.json
│   └── tr_diario.json
└── docs/
    └── img/
```

## Desafíos y aprendizajes

- Configurar correctamente las rutas del storage y los datasets de Data Factory (ADF necesita el nombre exacto del archivo, si no puede interpretar una carpeta como archivo).
- Pasar de una conexión manual con la clave del storage a **Managed Identity + Unity Catalog** al automatizar: hubo que configurar permisos sobre el workspace, el cluster y la External Location.
- Validar la calidad de los datos **antes** de agregar, para que los registros inválidos nunca lleguen a la capa analítica.

## Mejoras futuras

- Guardar los registros rechazados en una tabla específica para tener trazabilidad
- Tests automatizados para las reglas de calidad
- Monitoreo y alertas ante fallos
- Triggers basados en eventos (ejecutar al llegar un archivo nuevo)
- Cargas incrementales y optimización para mayores volúmenes
- Más fuentes de datos e indicadores de negocio

## Autor

**Benjamín Briolini** · [GitHub](https://github.com/Benjabrio)

Diplomatura en Data Engineering · ICARO
