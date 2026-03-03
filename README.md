# Diseño de un Módulo de Shelf Selection (SS) para un Sistema Robótico Goods-to-Person (G2P)

Este repositorio documenta el desarrollo de una solución de optimización para el módulo de **Shelf Selection (SS)**, un componente crítico de un sistema **Warehouse Execution System (WES)** que opera bajo la tecnología **Goods-to-Person (G2P)**. La implementación utiliza **Programación Lineal Entera (PLE)** para orquestar de manera eficiente el movimiento de estanterías móviles mediante robots autónomos en un centro de distribución.

Este proyecto fue desarrollado en el marco de la materia *Optimización Aplicada a la Distribución Logística* (FCEN–UBA, 2C 2025).

---

## 1. El Problema de Shelf Selection (SS)

En los centros de distribución modernos, la arquitectura de software se organiza en tres capas: **WMS** (gestión de inventario), **WES** (optimización de flujos en tiempo real) y **RCS** (control físico de robots). Mi trabajo se centra en el módulo SS dentro del WES para el proceso de *Outbound*.

### Objetivo del Módulo
La función principal que diseñé para el SS es decidir, cada 5 minutos, qué estanterías (**racks**) deben ser transportadas a las estaciones de trabajo para procesar las órdenes pendientes. El sistema debe:
* Seleccionar un conjunto de órdenes del backlog y asignarlas a caras de racks específicos.
* Maximizar la **densidad de picking** (cantidad de ítems extraídos por cada vez que un robot presenta una cara).
* Minimizar el esfuerzo de movimiento de los robots.

---

## 2. Criterios de Optimización

El algoritmo optimiza los siguientes criterios siguiendo un orden de prioridad estricto:

1.  **Maximizar "Picks Per Face":** Agrupar la mayor cantidad de órdenes en la menor cantidad de caras de racks para reducir los viajes de los robots.
2.  **Minimizar Costos de Movimiento:** Modelado de tres estados térmicos para los racks:
    * **Rack Frío (Costo 5):** Traer un rack que no ha sido usado recientemente.
    * **Rack Tibio (Costo 3):** Reutilizar un rack que se encuentra en un área de almacenamiento rápido.
    * **Rack Caliente (Costo 1):** Rotar un rack que ya está en uso en el mismo ciclo para presentar otra de sus caras.
3.  **Cumplimiento de SLA:** Priorizar órdenes con fechas de vencimiento (*due date*) próximas para evitar penalizaciones críticas.

---

## 3. Formulación Matemática (PLE)

Para resolver este problema de optimización combinatoria, formulé un modelo de **Programación Lineal Entera Binaria** utilizando la librería **PuLP**.

### 3.1. Variables de Decisión
* $A_{o,c,r,p} \in \{0, 1\}$: Vale $1$ si la orden $o$ se asigna a la cara $c$ del rack $r$ para el producto $p$.
* $B_{c,r} \in \{0, 1\}$: Vale $1$ si se utiliza la cara $c$ del rack $r$.
* $F_r, T_r, C_r \in \{0, 1\}$: Indican si el rack $r$ utilizado es frío, tibio o caliente, respectivamente.

### 3.2. Función Objetivo
El modelo busca maximizar el beneficio de las asignaciones netas de costos y penalizaciones:

$$\max \quad 4 \sum_{o \in O} \sum_{c,r,p} A_{o,c,r,p} + 8 \sum_{o' \in O^{TM}} \sum_{c,r,p} A_{o',c,r,p} - \sum_{c,r} B_{c,r} - 5 \sum_{r} F_r - 3 \sum_{r} T_r - \sum_{r} C_r - \sum_{o} Fec_o (1 - \sum_{c,r,p} A_{o,c,r,p})$$

Donde:
* $O^{TM}$ son las órdenes no procesadas por el Task Manager en el ciclo anterior, a las que asigno mayor prioridad.
* $Fec_o$ es una función de penalización por SLA que crece exponencialmente conforme se acerca el vencimiento:
    $$Fec_o(t_o) = \begin{cases} M_{\le 5} & \text{si } 0 < t_o \le 5 \text{ min} \\ \alpha \exp\left(-\frac{t_o-5}{w}\right) & \text{si } t_o > 5 \text{ min} \end{cases}$$

---

## 4. Arquitectura y Flujo de Procesos

Propuse una arquitectura modular basada en microservicios comunicados vía **REST** o **Kafka** para garantizar la asincronía y robustez del sistema.

### Flujo de Trabajo
1.  **Invocación:** El WES llama al SS cada 5 minutos proporcionando el backlog, órdenes pendientes del Task Manager y la capacidad $N$.
2.  **Preparación de Ventana:** Se filtran las órdenes creadas en el intervalo actual y se suman las no procesadas anteriormente.
3.  **Resolución:** El SS ejecuta el modelo PLE y devuelve un "pool de tareas" óptimo agrupado por cara de rack.
4.  **Actualización:** El sistema descuenta el stock del archivo central (`stock_v2.json`) y actualiza el estado térmico de los racks para la próxima corrida.

---

## 5. Detalles de la Implementación

Para que el modelo fuera viable en un entorno productivo con tiempos de respuesta exigentes (ventanas de 5 min), incorporé optimizaciones clave:

### 5.1. Preprocesamiento e Indexación
* **Poda de Racks:** El algoritmo ignora racks que no contengan productos requeridos por las órdenes del pool actual.
* **Renombre Temporal:** Traduzco IDs largos a índices enteros correlativos (0..R-1) para reducir el consumo de memoria del solver.

### 5.2. Creación Selectiva de Variables
Evité la creación de un tensor completo de 4 dimensiones. El algoritmo **solo instancia variables** $A_{o,c,r,p}$ para las combinaciones que tienen stock disponible y lógica funcional en la iteración actual.

### 5.3. Control Térmico Persistente
Mantengo un vector global (`estado_racks`) que persiste entre corridas:
* **Uso activo:** Si un rack se utiliza, su estado se setea a 0.
* **Enfriamiento:** Si no se usa, el valor decrementa hasta un mínimo de -3.
* **Clasificación:** Un rack es "Frío" si su estado es -3 y "Tibio" si está entre 0 y -2.

---

## 6. Métricas y KPIs Obtenidos

Validé la performance del algoritmo mediante una simulación de 24 horas de operación continua.

| Métrica (KPI) | Resultado | Descripción |
| :--- | :--- | :--- |
| **Picks Per Face** | **8.0** | Eficiencia en la agrupación de órdenes por cara. |
| **Uso de Racks Fríos** | **37.06%** | Proporción de racks nuevos versus racks reutilizados. |
| **Mínimo Margen SLA** | **-0.07 min** | Peor caso detectado (vencimientos casi inmediatos). |
| **Tiempo de Procesamiento** | **28.77 seg** | Latencia promedio del solver por ciclo de 5 minutos. |

---

## 7. Conclusiones

La implementación demuestra que es posible realizar asignaciones óptimas en pocos segundos, garantizando el cumplimiento de los SLAs y maximizando la densidad de picking. El sistema es robusto ante variaciones en la capacidad $N$ y prioriza correctamente las órdenes críticas.
