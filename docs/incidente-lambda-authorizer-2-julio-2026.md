# Reporte de Incidente — Lambda Authorizer `ciencuadras-authorizer-validate-token-prod`

## Información General

| Campo | Valor |
|-------|-------|
| Fecha | 2 de julio de 2026 |
| Inicio | 11:00 COL |
| Resolución | 11:34 COL |
| Duración | ~34 minutos |
| Severidad | P1 — Indisponibilidad total del portal |
| Servicio | `ciencuadras-authorizer-validate-token-prod` |
| Control de cambio asociado | MDSB-1035680 (status: Cambio Sin Éxito) |

---

## Descripción

La Lambda Authorizer de producción — componente transversal que valida la autenticación de todas las peticiones al portal ciencuadras.com — comenzó a fallar masivamente con `Runtime.ExitError` durante la fase de inicialización (cold start). Al ser el punto de entrada obligatorio del API Gateway, la falla causó un efecto cascada que dejó el portal completamente inaccesible.

---

## Impacto

| Servicio | Reducción de tráfico |
|----------|---------------------|
| API Gateway (`api-backend.ciencuadras.com`) | -99.8% |
| `ciencuadras-resultados-ms` | -93% |
| `ciencuadras-publicaciones-ms` | -96% |
| `ciencuadras-inmuebles-ms` | -93% |
| `ciencuadras-zona-privada-ms` | -89% |
| `ciencuadras-leads-ms` | -84% |
| `ciencuadras-agentes-ia-ms` | -86% |

**Errores registrados:** ~248,990 `Runtime.ExitError` + ~250,435 warnings de Datadog Extension.

---

## Causa Raíz

Falla en la inicialización de la Datadog Lambda Extension dentro del runtime `nodejs:24.v44`. El proceso Node.js terminaba abruptamente durante la fase INIT sin llegar a ejecutar el handler de la función. No se registraron deploys ni cambios de configuración en la lambda previo al incidente. La falla se atribuye a una inestabilidad transitoria en la interacción entre la extensión de Datadog y el entorno de ejecución de AWS Lambda durante un reciclaje masivo de execution environments.

---

## Evidencias

### 1. CloudWatch Logs — Error durante inicialización

```
INIT_START Runtime Version: nodejs:24.v44
EXTENSION Name: datadog-agent State: Ready
INIT_REPORT Init Duration: 509.62 ms  Phase: init  Status: error  Error Type: Runtime.ExitError
Exiting: failure, deadline: 1783008086049
```

### 2. Datadog APM — Caída y recuperación del tráfico

- Tráfico durante incidente (11:00-11:30): **3 requests** en API Gateway
- Tráfico post-recovery (11:30-12:00): **1,449 requests** en API Gateway
- Authorizer post-recovery: **15,971 requests** exitosos (status: ok)

### 3. GitHub Actions — Sin deploys a producción previo al incidente

- Último deploy a `master` antes del incidente: **11 de mayo de 2026**
- Deploy de corrección (2 julio, 11:30 COL): `workflow_dispatch` a `master`
- Evidencia: [https://github.com/segurosbolivar/ciencuadras-authorizer-lambda/actions](https://github.com/segurosbolivar/ciencuadras-authorizer-lambda/actions)

### 4. Datadog Logs — Distribución de errores (11:00-11:30)

| Patrón | Cantidad | % |
|--------|----------|---|
| `INIT_REPORT ... Runtime.ExitError` | 248,990 | 12% |
| `REPORT ... Runtime.ExitError` | 248,432 | 12% |
| `DD_EXTENSION WARN Context not found` | 250,435 | 12% |
| `Formato de token no válido` | 228 | <1% |
| `Token caducado` | 42 | <1% |

### 5. Monitores Datadog disparados

- `api-backend.ciencuadras.com` cambio anormal en rendimiento (P1)
- `ciencuadras-agentes-ia-ms` cambio anormal en rendimiento (P1)
- `[Lambda] Exceso en la ejecución de código para extensiones` (P1)

---

## Línea de Tiempo

| Hora (COL) | Evento |
|---|---|
| 11:00 | Inicio de fallas `Runtime.ExitError` en cold starts |
| 11:00 | Portal ciencuadras.com inaccesible |
| 11:05 | Alertas P1 disparadas (OpsGenie) |
| 11:34 | Redeploy forzado vía CI/CD resuelve el incidente |
| 11:35 | Tráfico normalizado |

---

## Resolución

Se ejecutó un redeploy forzado de la función Lambda que recreó los execution environments con instancias limpias de la extensión de Datadog, resolviendo la falla de inicialización de manera inmediata.

---

## Recomendaciones

| # | Acción | Prioridad |
|---|--------|-----------|
| 1 | Implementar **Provisioned Concurrency** (mínimo 5-10 instancias) para eliminar dependencia de cold starts | Alta |
| 2 | Pinear la **Datadog Extension Layer** a una versión estable específica | Alta |
| 3 | Publicar **versiones inmutables** de la Lambda para historial y rollback | Alta |
| 4 | Crear **monitor específico** para `Runtime.ExitError` con umbral < 5 min | Media |
| 5 | Evaluar **canary deployment** para detectar problemas antes del 100% | Media |
