# Módulo de Facturación ARCA

Documentación para preparar la integración de la aplicación con ARCA mediante WSFEv1.

Arquitectura actual (V2): la aplicación y los comprobantes viven en Firebase; la clave digital de ARCA y la comunicación con ARCA viven en un gateway propio en una máquina de la empresa. El código está en el repositorio `Josen-Facturacion`.

## Guías

**[Guía ARCA de Homologación](GUIA_ARCA_HOMOLOGACION.md)** — trámites: cómo obtener y verificar el CUIT emisor y los puntos de venta, habilitar WSASS, generar el CSR, obtener y autorizar el certificado de homologación, y reunir la configuración fiscal necesaria.

**[Homologación: preparación y ejecución](HOMOLOGACION.md)** — checklist técnico para el desarrollador: ambiente, despliegue del gateway, verificación con `arcaHealthCheck` y casos de prueba antes de producción.

La homologación es un entorno de prueba. Los comprobantes obtenidos allí no tienen validez fiscal. La emisión en producción requiere certificados, autorizaciones y una implementación productiva separada.
