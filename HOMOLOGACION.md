# Preparar y ejecutar homologación ARCA

## Estado actual

El código V2 tiene el flujo de facturas, notas de crédito totales y reconciliación. La app y Firebase todavía necesitan la configuración del proyecto clonado; el gateway físico, el certificado de homologación, la CUIT emisora, los puntos de venta y la red de la fábrica también deben configurarse antes de probar contra ARCA.

La primera etapa debe usar exclusivamente el proyecto Firebase clonado y el ambiente `homologation`. No se despliegan reglas sobre la app original ni se emiten comprobantes productivos durante estas pruebas.

**Protección de credenciales:** no compartas la Clave Fiscal, la clave privada, el secreto HMAC ni archivos `.env` por chat, correo o repositorio. La clave privada se genera y queda en una máquina controlada por la empresa; Functions recibe solo el secreto HMAC mediante Secret Manager.

## 1. Preparar el Firebase clonado

El clon debe tener habilitados Authentication, Firestore, Storage y Cloud Functions, con la facturación y las cuotas necesarias para esos servicios. Confirma además que incluya o despliegue `loginKolmena` (codebase `default`) y las funciones ARCA (codebase `arca`). Una copia de Firestore por sí sola no configura esos servicios ni las aplicaciones Android/Web.

1. Anota el ID exacto del proyecto clonado. No uses el ID de producción.
2. En `app/firebase.clone.example.json`, completa una copia local llamada `app/firebase.clone.local.json` con el ID y la configuración Web del clon. Ese archivo está ignorado por Git.
3. Reemplaza `app/android/app/google-services.json` por el archivo descargado desde la aplicación Android del clon. La app compara el ID al arrancar y se detiene si no coincide con `FIREBASE_PROJECT_ID`.
4. Configura usuarios de prueba en Authentication y confirma que la cuenta administrativa reciba el claim `billingAdmin` en el clon.
5. Revisa las reglas existentes del clon. `firestore.rules` y `storage.rules` de este repositorio son referencia; no las despliegues sin integrarlas con las reglas de la app.

Para iniciar la app web desde `app/`:

```bash
flutter run -d chrome --dart-define-from-file=firebase.clone.local.json
```

Despliega las funciones al clon indicando su ID en cada comando. El comando no despliega reglas:

```bash
firebase deploy --project <ID_DEL_CLON> --only functions:default,functions:arca
```

Los archivos `functions/.env.<ID_DEL_CLON>` y `app/firebase.clone.local.json` son locales. No uses `firebase use` para seleccionar un proyecto predeterminado en esta copia.

## 2. Confirmar datos fiscales y puntos de venta

El titular o la contadora debe confirmar:

- CUIT emisora y condición frente al IVA ([descargar la constancia de inscripción](https://www.arca.gob.ar/buscador-consultas-publicas/) y [revisar qué significa el estado de la CUIT](https://www.arca.gob.ar/estadosadministrativosCUIT/estado-cuit/efectos.asp)).
- Razón social, domicilio fiscal/comercial, ingresos brutos y fecha de inicio de actividades.
- Tipo de comprobante habilitado y tratamiento fiscal de los productos.
- Punto de venta para Web Services de oficina y de San Pablo, ambos exclusivos del sistema (se crean en el portal de ARCA, servicio **Administración de Puntos de Venta**, tipo **"Factura electrónica - Web Services" (RECE)**; ver [referencia de comprobantes](https://www.arca.gob.ar/facturacion/regimen-general/comprobantes.asp)). Ejemplo: `0006` para oficina y `0007` para San Pablo.
- Cómo se identifica la sucursal en los remitos. San Pablo debe normalizarse a `san_pablo`; si el campo no llega, Functions usa el punto de venta por defecto.
- Documentación del receptor, datos de consumidor final y reglas de redondeo.

La configuración de ejemplo está en `functions/.env.example`. Completa `functions/.env.<ID_DEL_CLON>` con los valores confirmados, en particular `ARCA_ISSUER_*`, `ARCA_POINT_OF_SALE`, `ARCA_POINTS_OF_SALE` y `ORDER_BRANCH_FIELD`. Mantén `ARCA_ENVIRONMENT=simulated` hasta que el clon y el gateway estén aislados y verificados.

## 3. Obtener el certificado de homologación

El titular o una persona autorizada administra WSASS con su propia Clave Fiscal. La clave privada se genera en la máquina controlada por la empresa que alojará el gateway, dentro de `deploy/secrets/` (directorio ignorado por Git). Solo el CSR público se carga en WSASS. La clave privada no se copia a la computadora del desarrollador.

En la máquina de la fábrica, después de instalar el repositorio y Docker según [deploy/INSTALACION.md](deploy/INSTALACION.md):

```bash
cd /opt/arca-gateway/repo/deploy
mkdir -p secrets && chmod 700 secrets
openssl genrsa -out secrets/arca_private_key.pem 2048
chmod 600 secrets/arca_private_key.pem
openssl req -new -key secrets/arca_private_key.pem \
  -subj "/C=AR/O=<EMPRESA>/CN=facturacion-app/serialNumber=CUIT <CUIT_PERSONAL_WSASS>" \
  -out secrets/arca_homologacion.csr
```

La CUIT del subject debe seguir lo que WSASS requiere para la persona que genera el certificado (normalmente el CUIT/CUIL del titular que ingresa con su Clave Fiscal). Ejemplo completo:

```bash
openssl req -new -key secrets/arca_private_key.pem \
  -subj "/C=AR/O=Josen SA/CN=facturacion-app/serialNumber=CUIT 20123456789" \
  -out secrets/arca_homologacion.csr
```

No subas el `.key` a WSASS. Desde una estación autorizada, entra a [WSASS](https://www.arca.gob.ar/ws/WSASS/html/), carga el CSR y crea el certificado ([crear certificado](https://www.arca.gob.ar/ws/WSASS/html/crearcertificado.html), [manual completo](https://www.arca.gob.ar/ws/WSASS/WSASS_manual.pdf)). Guárdalo directamente en la carpeta segura del gateway como `secrets/arca_certificate.pem`. Después autoriza el alias para la CUIT emisora y el servicio `wsfe` ([crear autorización](https://www.arca.gob.ar/ws/WSASS/html/crearautorizacion.html), [ver servicios autorizados](https://www.arca.gob.ar/ws/WSASS/html/verservicios.html)).

Si se necesita transferir el CSR público a la estación que usa WSASS, eso no expone la clave privada. Evita transferir el `.key`; si la política interna exige generar el certificado en otra máquina, copia el archivo al gateway por un canal cifrado directo y elimina las copias temporales.

En la máquina del gateway, desde `gateway/`, instala dependencias si hace falta y verifica el par sin contactar a ARCA:

```bash
npm ci
npm run arca:check -- --cert ../deploy/secrets/arca_certificate.pem \
  --key ../deploy/secrets/arca_private_key.pem \
  --env homologation --cuit <CUIT_EMISORA>
```

No continúes si `arca:check` informa errores. Ajusta los permisos de los archivos para que solo el servicio del contenedor pueda leerlos. `deploy/secrets/` no debe aparecer en Git.

## 4. Instalar el gateway fijo en San Pablo

Sigue [deploy/INSTALACION.md](deploy/INSTALACION.md). La máquina debe arrancar automáticamente después de un corte, tener salida a Internet hacia ARCA y mantener disponible el gateway. Elige una ruta de entrada desde Firebase: IP pública con HTTPS o túnel. Con túnel no se abren puertos entrantes del router.

El secreto HMAC se genera en la máquina gateway y se conserva como secreto de la empresa. Carga el mismo valor en Secret Manager, usando el proyecto clonado de forma explícita:

```bash
firebase functions:secrets:set ARCA_GATEWAY_HMAC_SECRET --project <ID_DEL_CLON>
```

El CLI lo solicita de forma interactiva. No pegues el valor del secreto en el repositorio ni en esta conversación.

Configura `deploy/.env` y `functions/.env.<ID_DEL_CLON>` con el mismo ambiente y CUIT, y con la URL pública HTTPS del gateway. El certificado y la clave ARCA solo van en el gateway; Functions solo guarda `ARCA_GATEWAY_HMAC_SECRET`.

## 5. Desplegar y comprobar sin producción

1. Confirma que app, Firebase CLI, Android/Web, Authentication, Functions, Firestore y Storage apuntan al clon.
2. Mantén `ARCA_ENVIRONMENT=simulated` y valida el login, la pantalla Facturación y la configuración del clon.
3. Cambia a `homologation` tanto en gateway como en Functions. Despliega con `--project <ID_DEL_CLON>` y sin incluir `firestore` o `storage` en el comando.
4. Desde Facturación ejecuta `arcaHealthCheck` con una cuenta `billingAdmin`. Debe confirmar configuración, firma HMAC, FEDummy, WSAA, autorización WSFE y puntos de venta según lo que ARCA publique en homologación.
5. Emite comprobantes de prueba y confirma que aparezcan en la cuenta/entorno de homologación. La app debe rotularlos sin validez fiscal y no habilitar entrega como factura productiva.

### Casos de prueba

- Factura A/B/C según la condición confirmada, documento del cliente e importes.
- Sucursal de oficina y San Pablo; verificar que cada una use su punto de venta.
- Notas de crédito A/B/C totales, asociadas al comprobante original (`CbtesAsoc`), y su reflejo en Facturación.
- Doble toque, dos emisiones concurrentes y numeración correlativa.
- Resultado incierto durante `FECAESolicitar`: queda `review_required`; reconciliar antes de cualquier nuevo intento.
- Resultado `safe_to_retry`: queda `pending`; el usuario decide cuándo reintentar.
- Reiniciar gateway y confirmar que el ticket WSAA del volumen se reutilice.
- Apagar o desconectar el gateway y confirmar el manejo `pending`/`review_required` según el punto de corte.
- Revisar CAE, fecha, QR, comprobante asociado e impresión PDF de factura y nota de crédito.

El backend tiene pruebas con ARCA simulado y una prueba de contrato Functions↔gateway. Eso no sustituye la autorización real de WSASS ni una emisión de homologación desde la máquina de la fábrica.

## 6. Antes de habilitar producción

La homologación se considera completa cuando la conexión real pasa el diagnóstico, se recorren los casos anteriores y la contadora aprueba los datos e importes. Producción requiere certificado distinto, ambiente `production`, autorización y punto de venta confirmados, respaldo de secretos y un despliegue dirigido explícitamente al proyecto productivo. Esa etapa queda fuera de esta preparación.

## Anexo: enlaces y ejemplos para el estudio contable

Tareas del estudio con los enlaces oficiales correspondientes:

| Tarea | Dónde | Ejemplo esperado |
|---|---|---|
| Descargar constancia de inscripción | [Buscador ARCA](https://www.arca.gob.ar/buscador-consultas-publicas/) | CUIT activa, condición `IVA Responsable Inscripto` |
| Verificar habilitación a comprobantes electrónicos | [Emisión y autorización](https://www.arca.gob.ar/fe/emision-autorizacion/solicitud-autorizacion.asp) | Regimen general habilitado |
| Crear puntos de venta Web Services | Portal ARCA → **Administración de Puntos de Venta** (tipo RECE - Web Services) | `0006` oficina, `0007` San Pablo |
| Entrar a WSASS (solo homologación) | [WSASS](https://www.arca.gob.ar/ws/WSASS/html/) ([cómo adherirse](https://www.arca.gob.ar/ws/WSASS/WSASS_como_adherirse.pdf)) | Login con Clave Fiscal del titular |
| Crear certificado desde el CSR | [Crear certificado](https://www.arca.gob.ar/ws/WSASS/html/crearcertificado.html) | Alias `facturacion-app`, cert `.crt` descargado |
| Autorizar certificado a WSFE | [Crear autorización](https://www.arca.gob.ar/ws/WSASS/html/crearautorizacion.html) | Servicio `wsfe` autorizado para la CUIT emisora |
| Revisar servicios autorizados | [Ver servicios](https://www.arca.gob.ar/ws/WSASS/html/verservicios.html) | `wsfe` figura como autorizado |

Ejemplos concretos de cada paso (valores de prueba, no reales):

- CSR subject: `CN=facturacion-app, serialNumber=CUIT 20123456789, O=Josen SA, C=AR`.
- El certificado de homologación lo emite la autoridad de test (issuer `Computadores de Test`); el de producción es otro y se tramita por separado.
- Si el certificado lo genera el estudio contable en lugar del titular: la CUIT del CSR debe ser la del representado y el estudio necesita autorización de administrador de certificados para esa CUIT ([administrar representados](https://www.arca.gob.ar/registro-unico-tributario/funcionalidades/funcionalidades.asp)).
- Qué debe devolverle el estudio al desarrollador: CUIT emisora, condición fiscal, números de punto de venta (oficina/San Pablo) y el archivo `.crt` del certificado de homologación. **Nunca** la Clave Fiscal ni la clave privada.

Referencias técnicas de ARCA:

- [Documentación WSFEv1](https://www.arca.gob.ar/ws/documentacion/ws-factura-electronica.asp) y [arquitectura general de web services](https://www.arca.gob.ar/ws/documentacion/arquitectura-general.asp).
- [WSAA: cómo funcionan los tickets](https://www.arca.gob.ar/ws/documentacion/wsaa.asp) y [certificados](https://www.arca.gob.ar/ws/documentacion/certificados.asp).
- [Entorno de prueba de factura electrónica](https://www.arca.gob.ar/fe/ayuda/entorno-prueba.asp).

## Referencias

- [Plan V2](PLAN.md)
- [Estado de integración de la app](PLAN_INTEGRACION_APP.md)
- [Instalación del gateway](deploy/INSTALACION.md)
