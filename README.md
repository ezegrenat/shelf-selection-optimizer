# shelf-selection-optimizer

# Diseño de un Módulo de Shelf Selection (SS) para un Sistema Robótico Goods-to-Person (G2P)

[cite_start]Este repositorio documenta mi desarrollo de una solución de optimización para el módulo de **Shelf Selection (SS)**, un componente crítico de un sistema **Warehouse Execution System (WES)** que opera bajo la tecnología **Goods-to-Person (G2P)**[cite: 8, 13]. [cite_start]La implementación utiliza **Programación Lineal Entera (PLE)** para orquestar de manera eficiente el movimiento de estanterías móviles mediante robots autónomos en un centro de distribución[cite: 15, 116].

[cite_start]Este proyecto fue desarrollado en el marco de la materia *Optimización Aplicada a la Distribución Logística* (FCEN–UBA, 2C 2025)[cite: 99, 102].

---

## 1. El Problema de Shelf Selection (SS)

[cite_start]En los centros de distribución modernos, la arquitectura de software se organiza en tres capas: **WMS** (gestión de inventario), **WES** (optimización de flujos en tiempo real) y **RCS** (control físico de robots)[cite: 9, 10, 11, 12]. [cite_start]Mi trabajo se centra en el módulo SS dentro del WES para el proceso de *Outbound*[cite: 13].



### Objetivo del Módulo
[cite_start]La función principal que diseñé para el SS es decidir, cada 5 minutos, qué estanterías (**racks**) deben ser transportadas a las estaciones de trabajo para procesar las órdenes pendientes[cite: 14, 34]. El sistema debe:
* [cite_start]Seleccionar un conjunto de órdenes del backlog y asignarlas a caras de racks específicos[cite: 20].
* [cite_start]Maximizar la **densidad de picking** (cantidad de ítems extraídos por cada vez que un robot presenta una cara)[cite: 21].
* [cite_start]Minimizar el esfuerzo de movimiento de los robots[cite: 21].

---

## 2. Criterios de Optimización

[cite_start]El algoritmo optimiza los siguientes criterios siguiendo un orden de prioridad estricto[cite: 44]:

1.  [cite_start]**Maximizar "Picks Per Face":** Agrupar la mayor cantidad de órdenes en la menor cantidad de caras de racks para reducir los viajes de los robots[cite: 45].
2.  **Minimizar Costos de Movimiento:** Modelado de tres estados térmicos para los racks:
    * [cite_start]**Rack Frío (Costo 5):** Traer un rack que no ha sido usado recientemente[cite: 49].
    * [cite_start]**Rack Tibio (Costo 3):** Reutilizar un rack que se encuentra en un área de almacenamiento rápido[cite: 51].
    * [cite_start]**Rack Caliente (Costo 1):** Rotar un rack que ya está en uso en el mismo ciclo para presentar otra de sus caras[cite: 53, 54].
3.  [cite_start]**Cumplimiento de SLA:** Priorizar órdenes con fechas de vencimiento (*due date*) próximas para evitar penalizaciones críticas[cite: 55, 56].

---

## 3. Formulación Matemática (PLE)

[cite_start]Para resolver este problema de optimización combinatoria, formulé un modelo de **Programación Lineal Entera Binaria** utilizando la librería **PuLP**[cite: 118, 136].

### 3.1. Variables de Decisión
* [cite_start]$A_{o,c,r,p} \in \{0, 1\}$: Vale $1$ si la orden $o$ se asigna a la cara $c$ del rack $r$ para el producto $p$[cite: 148].
* [cite_start]$B_{c,r} \in \{0, 1\}$: Vale $1$ si se utiliza la cara $c$ del rack $r$[cite: 144].
* [cite_start]$F_r, T_r, C_r \in \{0, 1\}$: Indican si el rack $r$ utilizado es frío, tibio o caliente, respectivamente[cite: 150, 151, 153].

### 3.2. Función Objetivo
[cite_start]El modelo busca maximizar el beneficio de las asignaciones netas de costos y penalizaciones[cite: 162, 163]:

$$\max \quad 4\sum_{o \in O} \sum_{c,r,p} A_{o,c,r,p} + 8\sum_{o' \in O^{TM}} \sum_{c,r,p} A_{o',c,r,p} - \sum_{c,r} B_{c,r} - 5\sum_{r} F_r - 3\sum_{r} T_r - \sum_{r} C_r - \sum_{o} Fec_o(1 - \sum_{c,r,p} A_{o,c,r,p})$$

Donde:
* [cite_start]$O^{TM}$ son las órdenes no procesadas por el Task Manager en el ciclo anterior, a las que asigno mayor prioridad (coeficiente 8)[cite: 164, 165].
* [cite_start]$Fec_o$ es una función de penalización por SLA que crece exponencialmente conforme se acerca el vencimiento[cite: 172, 173]:
    [cite_start]$$Fec_o(t_o) = \begin{cases} M_{\le 5} & \text{si } 0 < t_o \le 5 \text{ min} \\ \alpha \exp\left(-\frac{t_o-5}{w}\right) & \text{si } t_o > 5 \text{ min} \end{cases}$$ [cite: 174, 175, 176]

### 3.3. Restricciones Principales
* [cite_start]**Capacidad (R1):** $\sum A_{o,c,r,p} \le N$ (no exceder las órdenes requeridas por el ciclo)[cite: 180, 183].
* [cite_start]**Asignación Única (R2):** $\sum_{c,r,p} A_{o,c,r,p} \le 1$ para cada orden $o$[cite: 182, 183].
* [cite_start]**Stock (R6):** $\sum_{o} A_{o,c,r,p} \le S_{c,r,p}$ (no superar las unidades disponibles en cada ubicación)[cite: 205, 209].
* [cite_start]**Activación (R5):** $\sum_{p} A_{o,c,r,p} \le B_{c,r}$ (una cara solo se activa si se le asigna una orden)[cite: 202, 205].

---

## 4. Arquitectura y Flujo de Procesos

[cite_start]Propuse una arquitectura modular basada en microservicios comunicados vía **REST** o **Kafka** para garantizar la asincronía y robustez del sistema[cite: 456, 458, 459].



### Flujo de Trabajo
1.  [cite_start]**Invocación:** El WES llama al SS cada 5 minutos proporcionando el backlog, órdenes pendientes del Task Manager y la capacidad $N$[cite: 436, 439].
2.  [cite_start]**Preparación de Ventana:** Se filtran las órdenes creadas en el intervalo actual y se suman las no procesadas anteriormente[cite: 437, 438].
3.  [cite_start]**Resolución:** El SS ejecuta el modelo PLE y devuelve un "pool de tareas" óptimo agrupado por cara de rack[cite: 442, 537].
4.  [cite_start]**Actualización:** El sistema descuenta el stock del archivo central (`stock_v2.json`) y actualiza el estado térmico de los racks para la próxima corrida[cite: 445, 525, 531].

---

## 5. Detalles de la Implementación

[cite_start]Para que el modelo fuera viable en un entorno productivo con tiempos de respuesta exigentes (ventanas de 5 min), incorporé optimizaciones clave[cite: 472, 517]:

### 5.1. Preprocesamiento e Indexación
* [cite_start]**Poda de Racks:** Al cargar el stock, el algoritmo ignora automáticamente cualquier rack que no contenga los productos requeridos por las órdenes del pool actual[cite: 478].
* [cite_start]**Renombre Temporal:** Traduzco los IDs largos de órdenes y productos a índices enteros correlativos (0..R-1), lo que reduce significativamente el consumo de memoria del solver[cite: 541, 542, 543, 544].

### 5.2. Creación Selectiva de Variables
Evité la creación de un tensor completo de 4 dimensiones para $A_{o,c,r,p}$. [cite_start]El algoritmo **solo instancia variables** para las combinaciones (orden, cara, rack, producto) que tienen stock disponible y lógica funcional en la iteración actual[cite: 518, 520, 521].

### 5.3. Control Térmico Persistente
[cite_start]Mantengo un vector global (`estado_racks`) que persiste entre corridas[cite: 526, 531, 565].
* [cite_start]Si un rack se utiliza, su estado se setea a 0[cite: 529, 563].
* [cite_start]Si no se usa, el valor decrementa hasta un mínimo de -3 (Frío)[cite: 529, 564].
* [cite_start]Esto permite diferenciar dinámicamente entre racks fríos (estado -3) y tibios (0, -1, -2)[cite: 530].

---

## 6. Métricas y KPIs Obtenidos

[cite_start]Validé la performance del algoritmo mediante una simulación de 24 horas de operación continua[cite: 570].

| Métrica (KPI) | Resultado | Descripción |
| :--- | :--- | :--- |
| **Picks Per Face** | **8.0** | [cite_start]Eficiencia en la agrupación de órdenes por cara[cite: 571, 574]. |
| **Uso de Racks Fríos** | **37.06%** | [cite_start]Proporción de racks nuevos versus racks reutilizados[cite: 575, 578]. |
| **Mínimo Margen SLA** | **-0.07 min** | [cite_start]Peor caso detectado (órdenes con vencimiento casi inmediato)[cite: 579, 582]. |
| **Tiempo de Procesamiento** | **28.77 seg** | [cite_start]Latencia promedio del solver para encontrar la solución óptima[cite: 584, 586]. |

---

## 7. Conclusiones

[cite_start]La implementación demuestra que es posible realizar asignaciones óptimas en pocos segundos, garantizando el cumplimiento de los SLAs y maximizando la densidad de picking[cite: 589]. [cite_start]El sistema es robusto ante variaciones en la capacidad $N$ y prioriza correctamente las órdenes críticas[cite: 590].
