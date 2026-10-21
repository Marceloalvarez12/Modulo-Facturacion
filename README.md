# Módulo de Facturación ARCA

Documentación para preparar la integración de la aplicación con ARCA mediante WSFEv1.

Arquitectura actual (V2): la aplicación y los comprobantes viven en Firebase; la clave digital de ARCA y la comunicación con ARCA viven en un gateway propio en una máquina de la empresa. El código está en el repositorio `Josen-Facturacion`.

## Guía

**[HOMOLOGACION.md](HOMOLOGACION.md)** — preparación y ejecución de la homologación: datos del emisor, puntos de venta, trámites de WSASS y certificado, despliegue del gateway, verificación con `arcaHealthCheck` y casos de prueba antes de producción.

La homologación es un entorno de prueba. Los comprobantes obtenidos allí no tienen validez fiscal. La emisión en producción requiere certificados, autorizaciones y una implementación productiva separada.
