# Observabilidad con OpenTelemetry para un e-commerce multinube

Entrega académica parcial de los **puntos 1 y 2**: ADR y pipeline de telemetría para 12 microservicios en GCP GKE y AWS ECS.

## Documentos

- [Ver el PDF](OpenTelemetry_ADR_y_Pipeline.pdf) — 5 páginas: portada, ADR, pipeline y figura, configuración del Collector, referencias.
- [Descargar el Word editable](OpenTelemetry_ADR_y_Pipeline.docx) — **este es el archivo con el que trabaja el grupo.**
- [Descargar el paquete de documentos y anexos](Entrega_OpenTelemetry_Puntos_1_y_2.zip)
- [Guía para que el grupo complete los puntos 3, 4 y 5](GUIA_PARA_COMPANEROS.md)

El documento incluye citas y referencias en formato APA 7: márgenes de 2,54 cm, doble espacio, Times New Roman 12, sangría francesa en las referencias y paginación.

## Pendiente en la portada

La portada lleva el título, el subtítulo, los cinco integrantes y la fecha. **Faltan tres datos** que debe completar el grupo antes de entregar:

- Institución educativa
- Asignatura
- Nombre del docente

## Integrantes

- Carlos Arturo Salazar Castaneda
- Juan Sebastian Buitrago Romero
- Juan Jose Velez Alvarez
- Jhonatan Winston Sotelo
- Santiago Eduardo Muñoz Castillo

## Contenido desarrollado

1. **ADR-001, formato de Michael Nygard:** estado, contexto, opciones consideradas, decisión y consecuencias.
2. **Pipeline de instrumentación:** aplicación → OTel SDK → OTel Collector → backends, con receivers, processors, exporters y extensión de salud.

La propuesta utiliza gateways por nube. Las métricas y los logs se envían al backend de su nube; las trazas de GKE y ECS convergen en Jaeger.

| Señal | GCP | AWS |
| --- | --- | --- |
| Métricas | Cloud Monitoring mediante googlecloud | CloudWatch mediante awsemf |
| Logs | Cloud Logging mediante googlecloud | CloudWatch Logs mediante awscloudwatchlogs |
| Trazas | Jaeger compartido mediante OTLP | Jaeger compartido mediante OTLP |

## Presupuesto de páginas

La actividad pide **un documento de 6 a 8 páginas con los cinco puntos**. Los puntos 1 y 2 ocupan **3 páginas de cuerpo** (más portada y referencias, que son compartidas). Reparto previsto del documento final:

| Página | Contenido |
| --- | --- |
| 1 | Portada |
| 2 | ADR-001 (punto 1) |
| 3 | Pipeline y figura (punto 2) |
| 4 | Configuración del Collector (punto 2) |
| 5 | Punto 3: auto vs. manual |
| 6 | Punto 4: sampling y SLIs |
| 7 | Punto 5: taxonomía semántica |
| 8 | Referencias |

Los YAML completos van como **anexo digital**, no dentro del límite de páginas. Si el docente exige que los anexos cuenten, hay que confirmar la distribución con él.

## Arquitectura

![Arquitectura de instrumentación con componentes reales del Collector](arquitectura.png)

[Editar el diagrama en Mermaid](arquitectura.mmd). La fuente Mermaid reproduce el mismo diagrama que el PNG.

## Configuración del Collector

| Archivo | Uso |
| --- | --- |
| [collector-base.yaml](collector-base.yaml) | Recepción OTLP con mTLS, processors y exportación de trazas |
| [collector-gcp.yaml](collector-gcp.yaml) | Perfil de métricas y logs para GCP |
| [collector-aws.yaml](collector-aws.yaml) | Perfil de métricas y logs para AWS |
| [collector-gcp-completo.yaml](collector-gcp-completo.yaml) | Base y perfil GCP combinados |
| [collector-aws-completo.yaml](collector-aws-completo.yaml) | Base y perfil AWS combinados |
| [sdk-ejemplo.env](sdk-ejemplo.env) | Variables de ejemplo del SDK en la aplicación |

Versión de referencia: **Collector Contrib 0.160.0**. Especificación consultada: **OpenTelemetry 1.61.0**. Las Semantic Conventions se versionan por separado.

En el documento el YAML base aparece en estilo de flujo compacto para caber en una página; los archivos de este repositorio conservan el formato extendido y son los que se usan en el despliegue. Ambos son equivalentes.

Con el binario correspondiente, certificados y variables configurados:

~~~sh
otelcol-contrib validate --config=collector-gcp-completo.yaml
otelcol-contrib validate --config=collector-aws-completo.yaml
~~~

Para arrancar, elegir el archivo completo de la nube correspondiente o combinar la base con un único perfil. No cargar ambos perfiles en el mismo Collector.

Se verificaron la sintaxis YAML, las referencias entre componentes y la presentación del PDF. Las pruebas con el binario del Collector, autenticación cloud y carga a 10k RPS quedan pendientes. Consultar [VALIDACION.txt](VALIDACION.txt).

## Continuidad del equipo

- [ ] Completar institución, asignatura y docente en la portada.
- [ ] **Punto 3:** comparar instrumentación automática y manual: tiempo de desarrollo, CPU/memoria y granularidad.
- [ ] **Punto 4:** justificar head-based o tail-based sampling para 10k RPS y analizar su impacto en SLIs.
- [ ] **Punto 5:** definir atributos semánticos para HTTP, DB, RPC y eventos de negocio.
- [ ] Integrar las secciones y comprobar que el documento final no pase de 8 páginas.

El YAML todavía no establece una política de sampling. Si el grupo elige tail sampling, deberá reunir los spans de cada traza en el mismo sampler, incluso cuando crucen nubes. Los SLIs deben basarse en métricas de todas las solicitudes elegibles, independientemente del muestreo de trazas.

La [guía de continuidad](GUIA_PARA_COMPANEROS.md) detalla el contenido de cada punto, las fórmulas de volumen, los criterios de medición y las condiciones de integración.

## Fuentes principales

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Michael Nygard: Documenting architecture decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [Google Cloud: Instrument for Cloud Trace](https://docs.cloud.google.com/trace/docs/setup)

El enlace del material del curso (`cloud.google.com/trace/docs/setup/opentelemetry`) redirige a `docs.cloud.google.com` y devuelve 404; se cita la página vigente de esa misma guía.

Las referencias completas y las fuentes de los exportadores están en el documento.
