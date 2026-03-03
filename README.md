# Diseño de un Módulo de Shelf Selection (SS) para un Sistema Robótico Goods-to-Person (G2P)

Implementación del módulo de Shelf Selection (SS) para un sistema "Goods-to-Person" (G2P) de fulfillment. El modelo asigna órdenes de picking a caras de racks móviles maximizando la densidad de picking, minimizando el costo de movimiento de robots y priorizando el cumplimiento de SLAs. Lo resuelvo con programación lineal entera ninaria usando PuLP + CBC.  

<img width="906" height="595" alt="imagen" src="https://github.com/user-attachments/assets/d0786334-9d70-40a3-a287-518d1d8d6d57" />

>   Notar que, además de modelar este problema con Programación Lineal Entera, podría ser muy interesante abordarlo con metaheurísticas. De esta forma se podría sacrificar optimalidad en las asignaciones a cambio de velocidad, algo que podría ser ideal en casos específicos del negocio.

> Este proyecto fue hecho como trabajo final para la materia "optimización en logística" (FCEN-UBA. 2c 2025). Los datos -tanto el backlog como el stock) si bien son representativos de un caso real son ficticios y fueron provistos por la cátedra. 

---

## Resumen del problema

En un centro de distribución con tecnología G2P, robots autónomos traen racks completos hasta las estaciones de trabajo donde los operarios realizan el picking. El software se organiza en tres capas: WMS (inventario), WES (optimización en tiempo real) y RCS (control de robots).

El módulo SS vive dentro del WES y se ejecuta cada 5 minutos. En cada ciclo recibe:
- Las órdenes creadas en esa ventana de 5 minutos.
- Las órdenes pendientes del ciclo anterior (que el SS no pudo asignar).
- Las órdenes que el Task Manager (TM) no pudo procesar en el ciclo anterior (prioridad máxima).
- Un parámetro `N`: cantidad máxima de órdenes que el pool puede contener.
Y devuelve un pool de asignaciones: qué órdenes van a qué cara de qué rack.

El objetivo es hacer el mayor número de asignaciones posible concentrándolas en pocas caras de rack (más picks por cada vez que un robot mueve un rack), prefiriendo racks que ya estuvieron en uso recientemente (tibios) sobre los que llevan tiempo sin usarse (fríos), y evitando que se venzan órdenes con due date próximo.

---

### Variables de decisión, función objetivo y restricciones modeladas con programación lineal entera: 
están presentes en shelf-selection_formulacion.pdf

### Modelo térmico de racks: Es la forma que tiene el modelo para, mas allá de hacer asignaciones validas, favorecer asignaciones que den lugar a un tránsito ordenado de los robots en el deposito.
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
**`backlog.json`**                   : representación del backlog 
**`backlog.json`**                     : representación del backlog 
**`shelf-selection_formulacion.pdf`**                       : modelo de programación lineal entera utilizado para hacer las asignaciones.

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

Un poco mas de 390.000 órdenes generadas a lo largo de un día (2025-10-09). Cada orden pide exactamente 1 unidad de 1 producto. Los `cpt_hours` son los horarios de corte de promesas del día.

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

### Preparación de datos

El notebook lee `backlog.json` y `stock_v2.json` desde una subcarpeta `data/` relativa a donde esté el notebook. `stock_v2.json` es la copia de trabajo del stock que se va consumiendo a medida que avanza la simulación.

---

## Aspectos de implementación 
Notar que la creación de variables busca considerar solo combinaciones posibles para la ventana que se está trabajando, comparado a una "implementación naive" en la cual creamos todas las combinaciones posibles. La formulación naive instancia el tensor completo A[O][C][R][P], pero la mayoría de esas combinaciones nunca podrían valer 1: la orden o pide un producto específico, y ese producto solo existe en algunos racks y caras. Crear todas las variables desperdiciaría memoria y haría mucho mas lento al solver.

Por lo tanto, lo que se hizo fue construir primero el conjunto de tuplas (o, c, r, p) realmente factibles cruzando el backlog con el stock, y crear solo esas variables:

  A = {t: LpVariable(...) for t in indices}

En la misma línea, se podaron los racks que no contienen ningún producto requerido por la ventana actual, reduciendo R a solo los racks relevantes para cada ciclo.


