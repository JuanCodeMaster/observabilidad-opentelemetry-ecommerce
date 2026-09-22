# Guía para integrar el trabajo del grupo

## Qué está terminado
- Punto 1: ADR-001 con título, estado, contexto, opciones, decisión y consecuencias.
- Punto 2: pipeline de tres señales, figura, fuente Mermaid y configuración del Collector.
- PDF y Word: entrega parcial; no desarrollo de los cinco puntos.
- Portada: reemplazar nombres, institución, asignatura y docente.

## Archivos
- OpenTelemetry_ADR_y_Pipeline.docx y .pdf: documento principal.
- arquitectura.mmd: fuente editable Mermaid.
- arquitectura.png: figura para insertar en el documento.
- collector-base.yaml: recepción mTLS, processors y exportación de trazas.
- collector-gcp.yaml / collector-aws.yaml: perfiles que complementan la base.
- collector-gcp-completo.yaml / collector-aws-completo.yaml: configuraciones combinadas; usar una por nube.
- sdk-ejemplo.env: contrato orientativo del SDK.
- VALIDACION.txt: comprobaciones realizadas y pendientes.

## Acuerdos de integración
Son 12 servicios en total, no 12 por nube. No se inventó la distribución ni los lenguajes.
OTel API y SDK en aplicaciones; OTLP para transporte; gateways replicados por nube.
Métricas y logs quedan en su nube; trazas de ambas nubes convergen en Jaeger.
Cloud Monitoring recibe métricas; Cloud Logging recibe logs; Jaeger solo trazas.
Collector Contrib 0.160.0 para esta propuesta. Especificación consultada: 1.61.0. Semantic Conventions requiere una versión propia.
El YAML no decide sampling. Aplicar la decisión del punto 4 antes de producción.

## Punto 3: instrumentación automática vs. manual
Completar tabla: criterio | automática | manual | evidencia.
Criterios: tiempo de implementación, mantenimiento, CPU, memoria, arranque y granularidad.
Confirmar frameworks y bibliotecas soportadas. Proponer spans de negocio para checkout, inventario o pagos.
No inventar porcentajes de overhead: medir con mismas réplicas, dataset, carga y sampling, después del calentamiento.
Comparar tres ejecuciones: sin instrumentación, automática y automática + manual.
Medir CPU, RSS, p95/p99, spans/s y bytes/s; separar proceso y Collector.

## Punto 4: sampling y SLIs
Confirmar si 10k RPS son solicitudes de entrada o llamadas internas. Como hipótesis de cálculo: 10k solicitudes de entrada/s y una traza por solicitud; indicarlo.
- Trazas/s retenidas por head sampling = 10.000 * p.
- Spans/s antes de sampling = 10.000 * spans promedio por solicitud.
- No suponer que cada petición visita los 12 servicios.
- Medir bytes por span; no confundir tamaño serializado con memoria residente.
- Justificar p con errores esperados, presupuesto y carga.
- Head sampling no conoce necesariamente el resultado; lo descartado en origen no se recupera con tail sampling.
- Tail sampling requiere memoria, ventana de decisión y recepción de todos los spans de cada trace_id, incluso si cruzan nubes.
- Para tail distribuido, añadir exportación loadbalancing por trace_id a un pool común y tail_sampling antes de batch en ese pool. Ambos gateways deben compartir miembros y criterio de distribución.
- Un balanceador genérico por conexión no asegura afinidad de traza. No hacer tail sampling por separado en cada nube.
- No combinar head al 10 % con una promesa de conservar el 100 % de los errores.
- Obtener SLIs desde métricas de todas las solicitudes elegibles, no desde trazas sesgadas hacia errores/lentitud.
- Disponibilidad = exitosas elegibles / total elegible. Latencia: histograma de solicitudes en el borde elegido.
- No contar spans internos como nuevas compras. Vigilar pérdidas de métricas por exportación/reinicios.
Fuente: https://opentelemetry.io/docs/concepts/sampling/

## Punto 5: taxonomía semántica
Completar tabla: señal | convención/versión | atributo | tipo | ejemplo | obligatoriedad | cardinalidad | restricciones.
Recursos mínimos: service.name, service.namespace, service.version, service.instance.id, deployment.environment.name, cloud.provider, cloud.region.
Elegir nombres HTTP, DB y RPC de la versión consultada, sin mezclar convenciones antiguas/nuevas.
Definir eventos de negocio bajo un namespace propio y documentado.
No usar IDs de pedidos/usuarios como dimensiones de métricas; utilizar rutas parametrizadas.
Ampliar dimensiones awsemf de forma selectiva para ruta/resultado si lo requieren los SLIs; el perfil base solo usa servicio y entorno.
Comprobar histogramas, unidades, temporalidad y agregaciones de ambos backends.
Para SLIs globales, definir consulta/agregación entre nubes: sumar numeradores/denominadores y buckets compatibles; no promediar porcentajes o p99.
Fuente: https://opentelemetry.io/docs/specs/semconv/

## Despliegue
Los certificados y endpoints son un contrato, no secretos entregados.
Montar server.crt, server.key, ca.crt, jaeger-ca.crt, client.crt y client.key bajo /etc/otel/tls. Los SDK clientes también requieren certificado/clave.
JAEGER_OTLP_ENDPOINT = DNS privado:4317, certificado coincidente, ingreso Jaeger con mTLS y almacenamiento persistente.
GCP_PROJECT_ID = proyecto real. Workload Identity, roles/monitoring.metricWriter y roles/logging.logWriter.
AWS_REGION y COLLECTOR_ID = región y nombre único de Collector. Usar task role con permisos sobre los grupos previstos; PutLogEvents y creación/descripción según preaprovisionamiento.
Definir retención en Google, CloudWatch y Jaeger mediante IaC.
La cola de trazas mostrada es volátil; definir persistencia si la tolerancia a pérdidas lo exige.
AWS: temporalidad delta donde el SDK la soporte; no repartir conversiones acumulativas con estado sin afinidad de serie.
Límite inicial 1 GiB: no es una validación de capacidad para 10k RPS.
Este pipeline cubre señales de aplicación; la recopilación de nodos/pods/tareas se añade por separado.
Verificar trace_id/span_id en logs serializados en cada backend; evitar duplicación OTLP + agentes de stdout.

## Validación y arranque
Con binario 0.160.0, variables y certificados:
    otelcol-contrib validate --config=collector-gcp-completo.yaml
    otelcol-contrib validate --config=collector-aws-completo.yaml

Alternativa de arranque por capas:
    otelcol-contrib --config=collector-base.yaml --config=collector-gcp.yaml
    otelcol-contrib --config=collector-base.yaml --config=collector-aws.yaml

No cargar AWS y GCP simultáneamente.
Probar flujo entre nubes, error controlado, latencia artificial, pico de tráfico, caída de backend y reinicio de Collector. Medir pérdidas y correlación, no solo arranque.

## Documento final de 6–8 páginas
El PDF parcial contiene configuración y referencias para revisión; no es el desarrollo final de los cinco puntos.
Para integrar todo en ocho páginas: 1 portada; 2 ADR; 3 diagrama; 4 configuración resumida; 5 auto/manual; 6 sampling y SLIs; 7 taxonomía; 8 referencias.
Dejar YAML completos como anexos digitales si el docente lo permite. Si anexos cuentan en el límite, confirmar la distribución exigida.
Mantener márgenes de 2,54 cm, doble espacio en prosa, fuente legible, paginación y referencias APA. Código y figuras pueden usar espaciado simple.
El enlace de Google de la actividad no pudo recuperarse; se usó su guía oficial actual https://docs.cloud.google.com/trace/docs/setup.
