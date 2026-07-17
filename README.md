# Diagnóstico y Optimización FinOps en AKS: de sobreaprovisionamiento fantasma a capacidad real liberada

**Autor:** Carlos Amador López (Zorro)
**Stack:** Azure Kubernetes Service (AKS) · Terraform · OpenCost · Prometheus · Helm
**Repositorio base:** [cloud-aks-platform](https://github.com/carlozamlopez/cloud-aks-platform)
**Fecha del laboratorio:** Julio 2026

---

## Resumen ejecutivo

En un clúster de práctica en AKS, diagnostiqué que dos cargas de trabajo estaban reservando 550m de CPU y 512Mi de memoria mientras consumían en la práctica menos del 1% de esos recursos. Esa sobre-reserva bloqueaba la capacidad del nodo al punto de impedir el despliegue de un tercer servicio. Apliqué right-sizing basado en datos reales de consumo (no en estimaciones) y logré:

- **93.6% de reducción en CPU reservado** por las cargas de prueba (550m → 35m)
- **89% de reducción en memoria reservada** (512Mi → 56Mi)
- **12 puntos porcentuales menos de reserva total del nodo** (91% → 79%), *incluso con un tercer pod corriendo que antes estaba bloqueado*
- Capacidad de agendar nueva carga de trabajo sin agregar un solo nodo ni gastar más dinero

Todo el diagnóstico se hizo con herramientas gratuitas y open source (OpenCost + Prometheus), reproduciendo el mismo enfoque que usan equipos de FinOps en producción.

---

## 1. El problema (con contexto de industria)

Kubernetes no es más barato por default — es más barato solo si se opera con disciplina. Según el reporte 2026 de CAST AI, basado en telemetría directa de decenas de miles de clústeres en producción, el 69% de los clústeres sobreaprovisiona CPU (subiendo desde 40% el año anterior) y el 79% sobreaprovisiona memoria. La utilización promedio real de CPU en producción está en apenas 8%.

Esto no es un problema exclusivo de grandes empresas: es un patrón estructural. Los equipos de desarrollo piden más CPU/memoria de la que sus aplicaciones necesitan por seguridad ante fallas en producción, y ese margen de seguridad, multiplicado por cientos de pods, se convierte en gasto real sobre capacidad que nunca se usa — o peor, en un techo artificial que bloquea nueva carga de trabajo aunque el hardware esté prácticamente ocioso.

Este laboratorio reproduce ese problema desde cero, lo mide con datos propios, y aplica la corrección estándar de la industria (right-sizing basado en consumo real).

---

## 2. Infraestructura del laboratorio

Construida 100% con Terraform, siguiendo prácticas de bajo costo para operar como laboratorio personal:

| Componente | Configuración | Decisión de costo |
|---|---|---|
| AKS | Tier **Free**, 1 nodo `Standard_D2s_v3` | Control plane sin costo; evita el cargo de $72/mes del tier Standard, innecesario para un laboratorio |
| Log Analytics | `daily_quota_gb = 1` | Tope explícito para evitar facturación de ingesta sin límite |
| ACR | SKU Basic | Suficiente para imágenes de laboratorio |
| Networking | VNet con subredes segmentadas (management/workloads/dmz), NSGs con reglas explícitas | Refleja patrón de arquitectura real (Hub-and-Spoke simplificado), no solo un clúster plano |
| Hábito operativo | `az aks stop` / `az aks start` entre sesiones de trabajo | El clúster no corre 24/7; solo durante sesiones activas de laboratorio |

**Nota de aprendizaje incorporada:** durante el despliegue, Terraform falló porque Kubernetes 1.32 pasó a ser exclusivo de soporte LTS (de pago) en `eastus`. Se identificó la versión estándar soportada vigente (1.35.6) vía `az aks get-versions` y se corrigió sin habilitar LTS, evitando un costo adicional no planeado.

---

## 3. Fase 1 — Instrumentación y diagnóstico

### 3.1 Stack de observabilidad de costos

Se desplegó el mismo patrón que usan equipos reales de FinOps para clústeres pequeños/medianos, sin herramientas de pago:

- **Prometheus** (Helm chart oficial de la comunidad) como fuente de métricas, con persistencia mínima (PVC de 4Gi) para sobrevivir los ciclos de `stop`/`start` sin perder histórico.
- **OpenCost** (proyecto de la CNCF) como motor de asignación de costos por namespace/pod — el mismo motor sobre el que está construido Kubecost.
- **metrics-server** (ya incluido en AKS) como fuente de consumo real de CPU/memoria.

### 3.2 Cargas de trabajo de prueba

Se desplegaron 3 workloads deliberadamente sobreaprovisionados, diseñados para reproducir el patrón documentado por CAST AI/CNCF:

| Workload | Namespace | Propósito |
|---|---|---|
| `api-dummy-oversized` | workload-demo | Simula una API con requests sobredimensionados frente a una carga real mínima (nginx sirviendo contenido estático) |
| `batch-variable-load` | workload-demo | Simula un servicio que trabaja solo en ráfagas breves mientras reserva recursos 24/7 |
| `api-dummy-staging` | workload-staging | Testigo de bloqueo de capacidad — mismo patrón de requests, aislado para probar el efecto de la sobre-reserva sobre nueva carga de trabajo |

### 3.3 Evidencia del baseline

**Eficiencia general del clúster (OpenCost, ventana de 7 días):**

| Namespace | Eficiencia |
|---|---|
| Total del clúster | 11.5% |
| kube-system | 9.5% |
| opencost | 21.7% |

**Asignación de recursos a nivel de nodo, tras desplegar solo 2 workloads de prueba:**

```
Resource   Requests      Limits
--------   --------      ------
cpu        1732m (91%)   15152m (797%)
memory     2116Mi (29%)  20785440Ki (282%)
```

**Consumo real vs. reservado (kubectl top):**

| Pod | CPU solicitado | CPU real | % desperdicio | Mem solicitada | Mem real | % desperdicio |
|---|---|---|---|---|---|---|
| api-dummy-oversized | 300m | ~0m | >99% | 256Mi | 3Mi | 98.8% |
| batch-variable-load | 250m | 1m | 99.6% | 256Mi | ~0Mi | >99% |

**Consecuencia observada:** un tercer pod (`api-dummy-staging`) con requests idénticos (300m CPU / 256Mi) no pudo agendarse. El scheduler de Kubernetes lo rechazó con el evento:

```
0/1 nodes are available: 1 Insufficient cpu
```

Esta es la evidencia central del diagnóstico: **el clúster rechazó nueva capacidad de trabajo no por falta de cómputo real (uso real menor al 1% en los pods de prueba), sino por reservas de CPU sobredimensionadas que "llenaron" el nodo únicamente en el papel.**

---

## 4. Fase 2 — Intervención: right-sizing basado en datos

Con el consumo real medido (no estimado), se ajustaron los `requests`/`limits` de los dos deployments de prueba:

| Deployment | CPU antes | CPU después | Memoria antes | Memoria después |
|---|---|---|---|---|
| api-dummy-oversized | 300m request / 600m limit | 20m request / 100m limit | 256Mi / 512Mi | 32Mi / 64Mi |
| batch-variable-load | 250m request / 500m limit | 15m request / 150m limit | 256Mi / 512Mi | 24Mi / 64Mi |

### 4.1 Resultado

El pod `api-dummy-staging` — el mismo pod, sin recrearlo, sin tocar el nodo — pasó de `Pending` a `Running` únicamente por la liberación de espacio reservado.

**Asignación de recursos del nodo, ahora con 3 workloads corriendo (antes eran 2 y no cabía un tercero):**

```
Resource   Requests      Limits
--------   --------      ------
cpu        1517m (79%)   14902m (784%)
memory     1916Mi (26%)  20392224Ki (276%)
```

| Métrica | Antes (2 pods) | Después (3 pods) | Resultado |
|---|---|---|---|
| CPU reservado en nodo | 91% | 79% | -12 puntos porcentuales, con más carga corriendo |
| Memoria reservada | 29% | 26% | -3 puntos porcentuales |
| Pod workload-staging | Pending (bloqueado) | Running | Capacidad liberada sin agregar hardware |
| Reserva CPU combinada (2 pods de prueba) | 550m | 35m | **-93.6%** |
| Reserva memoria combinada (2 pods de prueba) | 512Mi | 56Mi | **-89%** |

### 4.2 Validación de persistencia

El resultado se validó en una segunda sesión, después de un ciclo completo de `az aks stop` → `az aks start`. Los valores de asignación de recursos se mantuvieron idénticos (1517m/79% CPU), confirmando que la corrección es estable y no un resultado circunstancial de una sola medición.

---

## 5. Aprendizajes técnicos del proceso

Más allá del resultado numérico, este laboratorio dejó lecciones operativas concretas, valiosas para cualquier equipo que opere Kubernetes con presupuesto ajustado:

1. **AKS vs. EKS tienen modelos de costo fundamentalmente distintos.** El control plane de AKS es gratuito en tier Free; el de EKS cobra $0.10/hora fijo *incluso si los nodos están apagados*. La estrategia de ahorro correcta no es la misma en ambas nubes: en AKS conviene `stop`/`start`, en EKS conviene `destroy`/`apply` cíclico.
2. **Persistencia mínima tiene costo real pero justificado.** Sin un PVC para Prometheus, cada ciclo de `stop`/`start` borra el histórico de métricas — inaceptable para un diagnóstico que requiere tendencia, no solo snapshots. Un disco de 4Gi (~$0.30-0.50 USD/mes) resuelve esto sin impacto significativo en presupuesto.
3. **Los `requests` no reflejan uso real — hay que medirlo, no asumirlo.** La diferencia entre "lo que parece prudente pedir" y "lo que la aplicación realmente consume" fue de más del 98% en ambos workloads de prueba, consistente con lo reportado a nivel de industria.
4. **El sobrecompromiso de `limits` es un problema aparte del de `requests`.** Aunque se corrigió el bloqueo de scheduling (ligado a requests), los `limits` del clúster siguen mostrando ~784% de sobrecompromiso — evidencia de que hay una capa adicional de optimización pendiente (ver Próximos pasos).

---

## 6. Decisión de arquitectura: ¿por qué no simplemente agregar un nodo?

> **Nota de honestidad metodológica:** a diferencia de las Fases 1 y 2 (medidas directamente en el laboratorio), esta sección es un **análisis de diseño** — un ejercicio de razonamiento arquitectónico sobre cómo sostener los resultados de right-sizing bajo tráfico real, sin haber desplegado la solución de Edge/CDN en este laboratorio. Se presenta como fundamento de decisión, no como resultado medido.

Una objeción legítima al right-sizing agresivo (20m CPU / 32Mi memoria) es: *"¿qué pasa cuando llega un pico real de tráfico?"* La respuesta ingenua — "agregar otro nodo 24/7 por si acaso" — es precisamente el tipo de decisión que perpetúa el sobreaprovisionamiento que este laboratorio buscó corregir. Devolvería el clúster al mismo patrón de reserva fantasma que documentamos en la Fase 1.

### 6.1 La alternativa: mover el buffer al perímetro (Edge), no al clúster

Para una carga de trabajo como `api-dummy-oversized` (contenido estático), la forma correcta de absorber picos de tráfico no es sobredimensionar los pods, sino colocar una capa de caché/CDN (Cloudflare, Azure Front Door, o similar) frente al clúster. El pico de conexiones concurrentes se resuelve en el perímetro de la red del proveedor; a los pods dentro de AKS solo les llega un flujo de tráfico ya filtrado y predecible — permitiendo mantener requests mínimos sin degradar el servicio.

### 6.2 Comparativo de costo (modelo, no medición)

Escenario hipotético: tráfico sostenido que un solo nodo `D2s_v3` con right-sizing ya no puede absorber cómodamente.

| Concepto | Estrategia A: nodo extra 24/7 | Estrategia B: Edge (caché/buffer) |
|---|---|---|
| Cómputo en AKS | ~$140 USD/mes (2× `Standard_D2s_v3` @ ~$70/mes c/u, eastus on-demand) | ~$70 USD/mes (1× `Standard_D2s_v3`, gracias al right-sizing ya validado) |
| Egress de datos | Variable, facturado por GB saliente desde Azure | Absorbido o descontado por el proveedor de Edge (varía según plan) |
| Costo del Edge/CDN | $0 | $0 a ~$20 USD/mes (planes base) |
| **Total estimado** | **~$140+ USD/mes** (sin contar egress) | **~$70-90 USD/mes** |

*Precios verificados en calculadoras públicas de Azure para `Standard_D2s_v3` on-demand en `eastus` (~$70/mes), julio 2026. El resto de las cifras (egress, planes de CDN) son estimaciones ilustrativas y deben cotizarse contra el proveedor específico antes de tomarse como definitivas.*

### 6.3 Simulación cuantitativa de un pico de tráfico

Para dimensionar el argumento anterior con números concretos, se modeló un escenario hipotético — **no ejecutado ni cargado sobre el clúster real**, sino calculado a partir de supuestos declarados explícitamente:

**Supuestos del escenario** (deben leerse como tal, no como mediciones):
- 50,000 usuarios concurrentes durante una ventana de 4 horas (ej. lanzamiento o campaña)
- 5 MB de assets estáticos por usuario → 250 GB de tráfico total
- Egress de Azure hacia internet a $0.08 USD/GB (tarifa estándar publicada)
- **Supuesto no verificado:** que absorber ese pico sirviendo directo desde AKS requeriría escalar de 1 a ~12 nodos vía HPA/Cluster Autoscaler — este número no se derivó de una prueba de carga real, sino de una estimación aproximada de conexiones por pod; se declara así para ser transparente sobre su naturaleza
- **Supuesto de industria:** 98% de cache hit ratio en la CDN, dentro del rango típico reportado por proveedores para contenido estático

**Escenario A — Absorber el pico escalando nodos en AKS:**

| Concepto | Cálculo | Costo |
|---|---|---|
| Nodos extra (11 adicionales × 4h, ~$0.10-0.11/hr c/u) | Estimado sobre tarifa on-demand de `D2s_v3` | ~$4.70 USD |
| Data egress (250 GB × $0.08) | Tráfico total servido directo desde Azure | $20.00 USD |
| **Total del pico** | | **~$24.70 USD** |

**Escenario B — Buffer en el Edge (CDN con 98% cache hit ratio):**

| Concepto | Cálculo | Costo |
|---|---|---|
| Nodos extra en AKS | Cero — el deployment se mantiene en sus requests mínimos (20m/32Mi) validados en la Fase 2 | $0 USD |
| Data egress residual (2% de 250 GB = 5 GB × $0.08) | Solo el tráfico que no fue absorbido por caché | $0.40 USD |
| Costo de la CDN | Capa gratuita/plan base típico para este volumen | $0 USD |
| **Total del pico** | | **~$0.40 USD** |

**Diferencia modelada:** ~98% menos costo por evento de pico en el escenario con Edge, bajo los supuestos declarados arriba. Este ahorro es adicional al 93.6% de reducción en reserva de CPU ya medido en la Fase 2 — uno es resultado de laboratorio, el otro es proyección de diseño.

### 6.4 El argumento central

Agregar un nodo 24/7 (o escalar agresivamente vía Cluster Autoscaler) para absorber picos ocasionales anula parcial o totalmente el 93.6% de reducción en reserva de CPU logrado en la Fase 2 — se estaría resolviendo un problema de picos con la misma estrategia (sobreaprovisionamiento) que causó el problema original. Mover el buffering al perímetro de red mantiene el clúster en su punto óptimo medido, y traslada la elasticidad a una capa diseñada específicamente para absorber picos de tráfico a menor costo marginal — según el modelo, del orden de 50-60 veces más barato por evento.

Agregar un nodo 24/7 para absorber picos ocasionales anula directamente el 93.6% de reducción en reserva de CPU logrado en la Fase 2 — se estaría resolviendo un problema de picos con la misma estrategia (sobreaprovisionamiento permanente) que causó el problema original. Mover el buffering al perímetro de red mantiene el clúster en su punto óptimo medido, y traslada la elasticidad a una capa diseñada específicamente para absorber picos de tráfico a menor costo marginal.

---

## 7. Próximos pasos (roadmap del portafolio)

- **Fase 3 — Multi-cloud:** replicar exactamente esta metodología en Amazon EKS, con las mismas herramientas (OpenCost + Prometheus), para demostrar que el enfoque no depende de un solo proveedor.
- **Fase 4 — Automatización de apagado programado:** implementar CronJobs que escalen a 0 réplicas los namespaces no-productivos fuera de horario laboral.
- **Fase 5 — Diagnóstico interactivo con IA:** herramienta que permita a cualquier equipo subir su propio export de OpenCost y recibir un reporte narrativo comparado contra benchmarks de industria, usando la API de Claude para generar el análisis en lenguaje de negocio.

---

## 8. Evidencia técnica adjunta

- `evidencia-pending-baseline.txt` — Evento de scheduler rechazando el pod por falta de CPU
- `evidencia-node-allocation-baseline.txt` — Asignación de recursos del nodo antes del right-sizing
- `evidencia-consumo-real-baseline.txt` — Salida de `kubectl top pods` con el consumo real
- `evidencia-node-allocation-after.txt` — Asignación de recursos del nodo después del right-sizing
- `evidencia-staging-after.txt` — Confirmación del pod pasando a estado `Running`
- Captura de pantalla — Dashboard de OpenCost mostrando eficiencia por namespace

---

## Fuentes citadas

- CAST AI, *2026 Kubernetes Cost Benchmark Report* — datos de sobreaprovisionamiento de CPU/memoria y utilización real en clústeres de producción.
- CNCF, *Cloud Native and Kubernetes FinOps Microsurvey 2024* — impacto de Kubernetes en costos de nube tras adopción.
