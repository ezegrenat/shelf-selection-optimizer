# Diseño de un Módulo de Shelf Selection (SS) para un Sistema Robótico Goods-to-Person (G2P)

Implemento el módulo de **Shelf Selection (SS)** para un sistema **Goods-to-Person (G2P)** de fulfillment como trabajo práctico de la materia *Optimización Aplicada a la Distribución Logística* (FCEN–UBA, 2C 2025). El modelo asigna órdenes de picking a caras de racks móviles maximizando la densidad de picking, minimizando el costo de movimiento de robots y priorizando el cumplimiento de SLAs. Lo resuelvo con **Programación Lineal Entera Binaria** usando PuLP + CBC.

> Los datos (backlog y stock) son ficticios y fueron provistos por la cátedra.

---

## El problema

En un centro de distribución con tecnología G2P, robots autónomos traen racks completos hasta las estaciones de trabajo donde los operarios realizan el picking. El software se organiza en tres capas: **WMS** (inventario), **WES** (optimización en tiempo real) y **RCS** (control de robots).

El módulo SS vive dentro del WES y se ejecuta cada 5 minutos. En cada ciclo recibe:
- Las órdenes creadas en esa ventana de 5 minutos.
- Las órdenes pendientes del ciclo anterior (que el SS no pudo asignar).
- Las órdenes que el **Task Manager (TM)** no pudo procesar en el ciclo anterior (prioridad máxima).
- Un parámetro `N`: cantidad máxima de órdenes que el pool puede contener.
Y devuelve un **pool de asignaciones**: qué órdenes van a qué cara de qué rack.

El objetivo es hacer el mayor número de asignaciones posible concentrándolas en pocas caras de rack (más picks por cada vez que un robot mueve un rack), prefiriendo racks que ya estuvieron en uso recientemente (tibios) sobre los que llevan tiempo sin usarse (fríos), y evitando que se venzan órdenes con due date próximo.

---

## Formulación del modelo

### Conjuntos e índices

| Símbolo | Descripción |
|---|---|
| `O` | Órdenes en la ventana actual |
| `O_TM` ⊆ O | Órdenes que el TM no procesó en el ciclo anterior |
| `C` | Caras de cada rack (1 a 4) |
| `R` | Racks con stock relevante para esta ventana |
| `P` | Productos referenciados en las órdenes de la ventana |

### Parámetros de entrada

| Símbolo | Descripción |
|---|---|
| `S[r][c][p]` | Stock del producto `p` en la cara `c` del rack `r` |
| `δ[o][p]` | 1 si la orden `o` requiere el producto `p`, 0 si no (cada orden pide un solo producto en este dataset) |
| `racksFrios` | Conjunto de racks con `estado_racks[r] == -3` (sin uso reciente) |
| `racksTibios` | Conjunto de racks con `estado_racks[r] ∈ {0, -1, -2}` |
| `N` | Máximo de órdenes procesables en el pool |
| `Fec_o` | Penalización por SLA de la orden `o` según minutos hasta su due date |

### Variables de decisión

| Variable | Tipo | Descripción |
|---|---|---|
| `A[o][c][r][p]` | binaria | 1 si la orden `o` se asigna a la cara `c` del rack `r` para el producto `p` |
| `B[c][r]` | binaria | 1 si la cara `c` del rack `r` es activada en este ciclo |
| `F[r]` | binaria | 1 si el rack `r` (frío) es utilizado |
| `T[r]` | binaria | 1 si el rack `r` (tibio) es utilizado |


### Función objetivo

Maximizar:

```
16 · Σ A[o][c][r][p]   (o ∈ O_TM)     ← órdenes TM pendientes (prioridad máxima)
+ 8 · Σ A[o][c][r][p]  (o ∉ O_TM)     ← resto de órdenes de la ventana
-     Σ B[c][r]                         ← penalización por caras usadas (incentiva PPF)
- 5 · Σ F[r]                            ← penalización por racks fríos usados
- 3 · Σ T[r]                            ← penalización por racks tibios usados
- (SLA_total - SLA_realizado)            ← penalización SLA por órdenes no asignadas
```

Donde `SLA_total = Σ Fec_o` y `SLA_realizado = Σ Fec_o · A[o]`, de forma que solo penaliza las órdenes que **no** fueron asignadas.

La función de penalización SLA:

```
Fec_o(t) = M                       si t ≤ 5 min  (vencimiento inminente → costo enorme)
         = α · e^(-(t-5)/w)        si t > 5 min
```

Con `M = 1e11`, `α = 0.5`, `w = 4` (parámetros en `fec_o()` del notebook principal).

### Restricciones

1. **Stock:** Para cada cara `c`, rack `r` y producto `p`, la cantidad total asignada no puede superar el stock disponible.
   ```
   Σ_o A[o][c][r][p] ≤ S[r][c][p]    ∀ c, r, p
   ```

2. **Una asignación por orden:** Cada orden solo puede ir a un lugar.
   ```
   Σ_{c,r,p} A[o][c][r][p] ≤ 1       ∀ o
   ```

3. **Capacidad del pool:** El total de asignaciones no puede superar `N`.
   ```
   Σ_{o,c,r,p} A[o][c][r][p] ≤ N
   ```

4. **Activación de cara:** Si una orden se asigna a una cara, esa cara queda activada.
   ```
   A[o][c][r][p] ≤ B[c][r]            ∀ o, c, r
   ```

5. **Activación de rack frío:** `F[r]` vale 1 si y solo si alguna cara del rack está activa (para `r ∈ racksFrios`).
   ```
   Σ_c B[c][r] ≤ 4 · F[r]
   F[r] ≤ Σ_c B[c][r]
   ```

6. **Activación de rack tibio:** Igual que la anterior pero para `r ∈ racksTibios` con `T[r]`.

### Modelo térmico de racks

Mantengo un vector global `estado_racks` de longitud 2089 (total de racks en el stock) que persiste entre ciclos:

- Se inicializa en `-3` (todos fríos).
- Si el rack fue usado en el ciclo: `estado_racks[r] = 0`.
- Si no fue usado: `estado_racks[r] = max(-3, estado_racks[r] - 1)`.
- Clasificación: frío si `== -3`, tibio si `∈ {0, -1, -2}`.

Esto modela el "enfriamiento" gradual: un rack que se deja de usar baja de tibio a frío en 3 ciclos (≈ 15 minutos).


### Archivos clave

**`optimizacion.ipynb`** — Notebook principal. Contiene:
- `fec_o(t, M, alpha, w)`: función de penalización SLA.
- `generar_ventana(desde, hasta)`: carga `backlog.json` y filtra las órdenes creadas en ese intervalo de 5 min. Devuelve un DataFrame con columna `m_vencimiento` (minutos hasta due date).
- `SS(N, df, estado_racks, total_no_procesadas_tm)`: arma y resuelve el modelo PLE. Actualiza `stock_v2.json` y `estado_racks`. Devuelve pool, DataFrames de asignadas/pendientes y KPIs.
- `simular_pendientes_tm(asignadas_df)`: simula aleatoriamente qué órdenes no procesó el TM.
- `WES()`: orquesta la simulación completa de 24 horas en ventanas de 5 minutos. Imprime KPIs al final.

**`backlog.json`**                     ← representación del backlog 
**`stock.json`**                       ← representación del stock 

---

## Flujo de ejecución

```
WES()
 └─ for cada ventana de 5 min (00:00 → 23:59):
      │
      ├─ generar_ventana()          # filtra órdenes del backlog por creation_date
      ├─ simular_pendientes_tm()    # simula qué órdenes no procesará el TM
      ├─ concatenar pendientes prev + TM prev + órdenes nuevas
      │
      └─ SS(N, ventana_df, estado_racks, n_tm_prev)
           │
           ├─ Mapeo item_id → entero 0..P-1
           ├─ Cargar stock_v2.json, podar racks sin productos relevantes
           ├─ Mapeo rack_name → entero 0..R-1 (solo racks con stock útil)
           ├─ Construir índices de A factibles (solo combinaciones (o,c,r,p) viables)
           ├─ Clasificar racks en fríos/tibios según estado_racks
           │
           ├─ Crear modelo PuLP (LpMaximize)
           │    ├─ Variables: A (sparse), B, F, T
           │    ├─ Restricciones 1–6
           │    └─ Función objetivo
           │
           ├─ Resolver con CBC (4 threads, timeout 120s, gap 1%)
           │
           ├─ Actualizar stock_v2.json (descontar productos pickeados)
           ├─ Actualizar estado_racks (frío/tibio para siguiente ciclo)
           │
           └─ Retornar:
                pool          → dict { (rack, cara): [orden_1, orden_2, ...] }
                asignadas_df  → DataFrame de órdenes asignadas
                pendientes_df → DataFrame de órdenes no asignadas (van al próximo ciclo)
                estado_racks  → vector actualizado
                asignadas     → int (cantidad asignada)
                ppf           → int (máx picks por face en este pool)
                venc_min      → float (minutos al vencimiento de la orden más urgente asignada)
                pct_frios     → float (% de racks fríos usados)
```

---

## Formato de inputs y outputs

### `backlog.json`

```json
{
  "metadata": { "cpt_hours": [6, 10, 14, 16, 19, 21, 22, 23.59] },
  "orders": [
    {
      "order_id": "ORD_013387_LXJY4YBS_000",
      "item_id":  "LXJY4YBSWCX3KBF",
      "quantity": 1,
      "creation_date": "2025-10-09T00:00:04.885077",
      "due_date":      "2025-10-09T21:00:00.885077"
    }
  ]
}
```

390.855 órdenes generadas a lo largo de un día (2025-10-09). Cada orden pide exactamente 1 unidad de 1 producto. Los `cpt_hours` son los horarios de corte de promesas del día.

### `stock.json` / `stock_v2.json`

```json
{
  "Rack_00001": {
    "Cara_1": [
      { "Inventory ID": "1U0KF6M5H964WNX", "Cantidad": 4, "Nivel": 2, "Posicion": 2 }
    ],
    "Cara_2": [],
    "Cara_3": [],
    "Cara_4": []
  }
}
```

2.087 racks, hasta 4 caras por rack, ~71.000 SKUs únicos. `Nivel` y `Posicion` son metadata de ubicación física dentro de la cara (no usados por el modelo).

### Output: `pool`

```python
{
  ("Rack_5", "Cara_1"): ["orden_1", "orden_3", "orden_7"],
  ("Rack_12", "Cara_1"): ["orden_2", "orden_4"],
}
```

Cada clave es un par `(rack, cara)` y el valor es la lista de órdenes asignadas a esa cara.

---

## Cómo correr el proyecto

### Requisitos

- Python 3.9+
- Jupyter Notebook o JupyterLab

```
pulp>=3.3.0
numpy
pandas
```

El solver CBC viene incluido con PuLP y no requiere instalación separada.

### Instalación

```bash
git clone <url-del-repo>
cd <nombre-del-repo>

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install pulp numpy pandas jupyter
```

### Preparación de datos

El notebook lee `backlog.json` y `stock_v2.json` desde una subcarpeta `data/` relativa a donde esté el notebook. `stock_v2.json` es la copia de trabajo del stock que se va consumiendo a medida que avanza la simulación.

```bash
mkdir data
cp backlog.json data/backlog.json
cp stock.json   data/stock_v2.json
```

> Si querés volver a correr la simulación desde cero, repetí el último `cp` para restaurar el stock original.

### Ejecución

```bash
jupyter notebook optimizacion.ipynb
```

Ejecutar todas las celdas en orden. La última celda (`WES()`) corre la simulación completa de 24 horas e imprime los KPIs al final.
