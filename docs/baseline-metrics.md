# Baseline Metrics — Fase 1: Diagnóstico

**Fecha de captura:** 2026-07-15
**Clúster:** aks-cloudaks-dev (AKS Free tier, 1 nodo Standard_D2s_v3, eastus)

## 1. Eficiencia general del clúster (OpenCost, últimos 7 días)

| Namespace | Eficiencia |
|---|---|
| Total del clúster | 11.5% |
| kube-system | 9.5% |
| opencost | 21.7% |
| prometheus-system | 0% |

## 2. Evidencia de sobreaprovisionamiento a nivel de nodo

Con solo 3 pods de carga de trabajo (`api-dummy-oversized`, `batch-variable-load` en workload-demo)
sumados a la infraestructura base, el nodo alcanzó:

- CPU reservado (requests): 91% del nodo
- CPU en límites (limits): 797% — sobrecompromiso de casi 8x
- Memoria reservada (requests): 29%

## 3. Consumo real vs. reservado (kubectl top)

| Pod | CPU solicitado | CPU real | % desperdicio | Mem solicitada | Mem real | % desperdicio |
|---|---|---|---|---|---|---|
| api-dummy-oversized | 300m | ~0m | >99% | 256Mi | 3Mi | 98.8% |
| batch-variable-load | 250m | 1m | 99.6% | 256Mi | ~0Mi | >99% |

## 4. Consecuencia observada

Un tercer pod (`api-dummy-staging`, namespace workload-staging) con requests idénticos
(300m CPU / 256Mi) no pudo agendarse:

Evento del scheduler: `0/1 nodes are available: 1 Insufficient cpu`

**Conclusión del diagnóstico:** el clúster rechazó nueva capacidad de trabajo no por falta
de cómputo real (uso real <1% en los pods de prueba), sino por reservas de CPU
sobredimensionadas que "llenaron" el nodo en el papel. Este es el mismo patrón que
CAST AI documentó en su reporte 2026: 69% de clústeres sobreaprovisionan CPU, con
utilización real promedio de apenas 8%.

## Evidencia adjunta
- evidencia-pending-baseline.txt
- evidencia-node-allocation-baseline.txt
- evidencia-consumo-real-baseline.txt
- Captura de pantalla: dashboard OpenCost (eficiencia 11.5%)

## 5. Resultado tras right-sizing (Fase 2)

| Métrica | Antes | Después |
|---|---|---|
| CPU reservado en nodo | 1732m (91%) | 1517m (79%) |
| Memoria reservada | 2116Mi (29%) | 1916Mi (26%) |
| Pod workload-staging | Pending (bloqueado) | Running |

**Cambios aplicados:**
- api-dummy-oversized: 300m→20m CPU, 256Mi→32Mi memoria (requests)
- batch-variable-load: 250m→15m CPU, 256Mi→24Mi memoria (requests)

**Resultado:** con un pod adicional corriendo (el de staging, previamente bloqueado),
la reserva total de CPU del nodo bajó 12 puntos porcentuales. Se demostró capacidad
de agendar más carga de trabajo sin agregar hardware, únicamente corrigiendo
requests mal calculados — la misma causa raíz que documenta el reporte 2026 de CAST AI.