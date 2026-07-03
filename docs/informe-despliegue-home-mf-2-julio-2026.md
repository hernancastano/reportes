# Informe Post-Despliegue — Home MF Ciencuadras (Optimización de Rendimiento)

## Resumen ejecutivo

El despliegue a producción del microfrontend Home de Ciencuadras presentó dos incidentes independientes durante el proceso de paso a producción:

1. Un fallo de compilación por una dependencia transitiva inestable de AWS (`@smithy/core`).
2. Un error HTTP 403 causado por la ausencia de ACL públicos en los objetos del bucket S3.

Ambos fueron resueltos de forma controlada gracias a un acompañamiento preventivo solicitado con anticipación.

- **Causa raíz:** Infraestructura (configuración de ACL en S3), no código.
- **Estado final:** Portal funcionando correctamente en producción.
- **Tiempo total de resolución:** 9 días calendario (desde la detección inicial hasta la resolución en producción).

---

## Cronograma de eventos

| # | Fecha y hora | Evento | Ticket | Estado |
|---|---|---|---|---|
| 1 | **Lun 23-jun, 10:20** | Detección del error 403 en ambientes bajos (develop/stage). Se reporta a infraestructura. | [MDSB-1065304](https://jirasegurosbolivar.atlassian.net/servicedesk/customer/portal/2/MDSB-1065304) | Pending |
| 2 | ~**Mié 25-jun** | Validación exitosa en ambiente **PRE** — sin error 403. Se confirma viabilidad de continuar. | — | — |
| 3 | **Jue 26-jun, 18:38** | Solicitud formal de Cambio Digital para el despliegue a producción. | [MDSB-1068593](https://jirasegurosbolivar.atlassian.net/servicedesk/customer/portal/2/MDSB-1068593) | Cambio Sin Éxito |
| 4 | **Mar 01-jul, 08:21** | Solicitud de acompañamiento Infraestructura para el despliegue. | [MDSB-1069965](https://jirasegurosbolivar.atlassian.net/servicedesk/customer/portal/2/MDSB-1069965) | In Progress |
| 5 | **Mar 01-jul, 08:22** | Solicitud de acompañamiento DevOps/BOX para el despliegue. | [MDSB-1069968](https://jirasegurosbolivar.atlassian.net/servicedesk/customer/portal/2/MDSB-1069968) | In Progress |
| 6 | **Mar 01-jul, ~22:00** | **Primer intento de despliegue a producción — FALLA.** El build no compiló por error de tipos en `@smithy/core`. | MDSB-1068593 | — |
| 7 | **Mié 02-jul, 09:26** | Solicitud de Cambio Digital para corregir la dependencia (`skipLibCheck: true` en tsconfig). | [MDSB-1071020](https://jirasegurosbolivar.atlassian.net/servicedesk/customer/portal/2/MDSB-1071020) | Cambio con Impacto |
| 8 | **Mié 02-jul, ~12:00** | **Segundo intento de despliegue a producción — BUILD OK, pero error 403 aparece.** | MDSB-1068593 | — |
| 9 | **Mié 02-jul, ~12:30** | Infraestructura ajusta los ACL en el bucket S3 de producción (acompañamiento ya activo). | MDSB-1069965 / MDSB-1069968 | — |
| 10 | **Mié 02-jul, ~13:00** | **Portal Home funcional en producción.** Error 403 resuelto. | — | OK |

---

## Análisis de los dos incidentes

### Incidente 1: Fallo de compilación — Dependencia `@smithy/core`

- **Causa:** La versión más reciente de `@smithy/core` (transitiva del AWS SDK) requiere tipos de Node 18+ (`node:stream/web`), mientras el proyecto usa Node 14. Como el `package-lock.json` no está versionado, cada `npm install` resuelve versiones nuevas.
- **Solución aplicada:** Se habilitó `skipLibCheck: true` en el `tsconfig.json` — opción estándar de Angular (activa por defecto desde v12) que omite la verificación de tipos de librerías de terceros sin afectar el runtime ni el bundle.
- **Observación crítica:** Este es un riesgo sistémico. Sin `package-lock.json` versionado, cualquier pipeline puede fallar de forma impredecible. Se recomienda versionar el lockfile o implementar una estrategia de pinning.

### Incidente 2: Error HTTP 403 — ACL de S3

- **Causa:** La action composite de despliegue (`segurosbolivar/devops-composite-actions/AWS/S3@master`) realiza el upload de archivos al bucket sin el flag `--acl public-read`. CloudFront no puede servir archivos con acceso restringido.
- **Contexto:** Infraestructura indicó inicialmente que este lineamiento de ACL solo aplicaba a ambientes bajos y que producción no debería verse afectado. Esta información resultó incorrecta — producción también tenía la restricción.
- **Solución aplicada:** El equipo de infraestructura ajustó manualmente los permisos públicos de los objetos en el bucket S3 de producción.
- **Observación crítica:** La validación en ambiente PRE no fue representativa del comportamiento en producción, lo que indica que PRE y PROD tienen configuraciones de infraestructura divergentes en cuanto a ACL/buckets. Esto invalida parcialmente la confiabilidad de PRE como puerta de calidad.

---

## Métricas clave

| Métrica | Valor |
|---|---|
| Tiempo detección → resolución completa | 9 días calendario |
| Tiempo entre primer y segundo intento de despliegue | ~14 horas |
| Tickets gestionados | 5 |
| Intentos de despliegue a producción | 2 |
| Incidentes bloqueantes | 2 (independientes) |
| Rollback requerido | No |
| Afectación a usuario final | Mínima (segundo intento se resolvió el mismo día) |

---

## Qué se hizo bien

1. **Acompañamiento preventivo:** Se solicitaron tickets de acompañamiento (infra + DevOps) antes del despliegue, lo que permitió reacción inmediata al aparecer el 403 en producción.
2. **Trazabilidad completa:** Cada paso del proceso quedó documentado en tickets con fechas, descripciones y responsables.
3. **Separación de problemas:** Los dos incidentes (dependencia vs. ACL) se gestionaron de forma independiente, sin mezclar causas ni soluciones.
4. **Validación en PRE:** Aunque PRE no replicó el fallo, el hecho de haberlo intentado demuestra un proceso ordenado.

---

## Puntos de mejora y recomendaciones

| # | Recomendación | Prioridad |
|---|---|---|
| 1 | **Versionar el `package-lock.json`** para evitar resoluciones impredecibles de dependencias en cada build. | Alta |
| 2 | **Solicitar a DevOps que la action composite de S3 incluya `--acl public-read`** por defecto o como parámetro configurable. El ticket MDSB-1065304 sigue en estado Pending — hacer seguimiento. | Alta |
| 3 | **Auditar la paridad de configuración entre PRE y PROD** en lo referente a permisos de bucket y distribución CloudFront. | Media |
| 4 | **Considerar usar OAI/OAC (Origin Access Identity/Control)** para que CloudFront acceda al bucket de forma privada, eliminando la dependencia de ACL públicos. | Media |
| 5 | **Documentar el runbook de despliegue** del Home MF con los pre-requisitos de infraestructura validados. | Baja |

---

## Conclusión

El problema nunca fue del código del equipo. La demora se originó por dos factores de infraestructura/tooling: una dependencia transitiva inestable y una configuración de permisos de bucket no aplicada en producción. El proceso fue metódico — se escaló a infraestructura, se validó en PRE, se programó acompañamiento y se resolvieron los dos problemas por separado.

La principal lección es que **el ambiente PRE no reflejaba fielmente la configuración de producción**, lo que generó una falsa seguridad. Se recomienda priorizar la paridad de entornos y el versionamiento del lockfile para evitar recurrencias.
