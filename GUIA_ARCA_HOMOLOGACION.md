# Guía para preparar la facturación ARCA en homologación

**Objetivo:** reunir la información fiscal y las autorizaciones necesarias para conectar el módulo de facturación con ARCA WSFEv1 en el entorno de prueba.

La aplicación existente no necesita rehacerse: el módulo se integra con funciones de Firebase que la app llama. Hoy el servicio arranca en modo simulado y está preparado para homologación y producción; se cambia con `ARCA_ENVIRONMENT`, sin tocar código. El CAE de homologación no tiene validez fiscal.

Esta guía es la versión detallada, con enlaces oficiales. El resumen para el titular y los comandos del desarrollador están en [HOMOLOGACION.md](HOMOLOGACION.md).

## 1. Datos del emisor

### CUIT que emitirá las facturas

La CUIT debe ser la del contribuyente que vende los productos y asume la emisión fiscal: persona o empresa titular del negocio. No es la CUIT del desarrollador ni la de quien crea el certificado técnico.

**Cómo obtenerla y verificarla:**

1. Pedir al titular o contador la **Constancia de inscripción** vigente.
2. También se puede buscar “Constancia de inscripción” en el sitio oficial de ARCA y consultar por CUIT.
3. Confirmar que la CUIT esté activa y revisar la condición frente a IVA y los impuestos/regímenes registrados. Si figura limitada o hay dudas sobre la autorización para emitir, resolverlo con el contador antes de probar.

**Lecturas oficiales:** [buscar la Constancia de inscripción](https://www.arca.gob.ar/buscador-consultas-publicas/) y [consultar qué significa el estado de la CUIT](https://www.arca.gob.ar/estadosadministrativosCUIT/estado-cuit/efectos.asp).

**Entregar al desarrollador:** CUIT del emisor (11 dígitos) y su condición fiscal confirmada por el contador. No enviar la Clave Fiscal.

### Punto de venta para Web Services

El punto de venta identifica la numeración usada por este sistema. Debe estar habilitado para facturación electrónica por **Web Services** y ser distinto de los que se usan para otros sistemas de emisión.

**Cómo revisarlo o darlo de alta:**

1. Ingresar a ARCA con la Clave Fiscal del titular o de una persona autorizada.
2. Abrir **Administración de puntos de venta y domicilios** (el nombre puede mostrarse como “Administración de Puntos de Venta y Domicilios”).
3. Seleccionar al contribuyente que va a emitir.
4. Entrar a **A/B/M de puntos de venta** y revisar los puntos ya existentes.
5. Si hay que crear uno, usar **Agregar**, elegir un número libre, seleccionar el domicilio y asociar el sistema de facturación de **Web Services**. No elegir “Comprobantes en línea” para el punto que usará este módulo.
6. Guardar y anotar el número exacto. Si la pantalla no ofrece una opción de Web Services o hay dudas sobre cuál corresponde al régimen del emisor, consultar al contador/soporte de ARCA antes de confirmarlo.

No crear un punto nuevo sin revisar antes los existentes. El punto de venta tiene que ser **exclusivo del módulo**: si otro sistema o el portal de ARCA emite con el mismo número, la numeración choca (error 10016). El diagnóstico `arcaHealthCheck` consulta los puntos registrados con `FEParamGetPtosVenta`, pero el alta se hace desde ARCA.

**Lecturas oficiales:** [solicitud de autorización y requisitos de los puntos de venta](https://arca.gob.ar/fe/emision-autorizacion/solicitud-autorizacion.asp) y [funciones del Registro Único Tributario](https://arca.gob.ar/registro-unico-tributario/funcionalidades/funcionalidades.asp). La pantalla y el recorrido pueden variar según el régimen del emisor; estas páginas explican el marco y dónde se administra el alta.

**Entregar al desarrollador:** número de punto de venta y confirmación de que figura habilitado para Web Services. En el backend se configura como `ARCA_POINT_OF_SALE`; no asumir que el valor de ejemplo/default `1` es el correcto.

## 2. Certificado de homologación (prueba)

El certificado permite que el backend se autentique ante WSAA. Para probar, se obtiene en **WSASS**, que es exclusivamente para testing/homologación. No sirve para facturar en producción.

### A. Habilitar el acceso a WSASS

1. Una **persona física** entra a ARCA con su propia CUIT y Clave Fiscal. El servicio WSASS no se adhiere con la CUIT de una empresa y no es delegable.
2. Si no aparece WSASS, usar **Administrador de Relaciones de Clave Fiscal** para adherir el servicio de homologación siguiendo la guía oficial de ARCA.
3. Ingresar a WSASS con esa Clave Fiscal.

**Guía paso a paso de ARCA:** [Cómo adherirse a WSASS (PDF)](https://www.arca.gob.ar/ws/WSASS/WSASS_como_adherirse.pdf). El [manual de WSASS](https://www.arca.gob.ar/ws/WSASS/html/) explica las opciones dentro del servicio.

La persona que administra WSASS puede ser el titular o alguien autorizado por la empresa. La CUIT de esa persona y la CUIT emisora pueden ser distintas: luego se autoriza el certificado para representar a la CUIT del emisor.

### B. Generar la clave privada y el CSR en el gateway de la empresa

La clave debe quedar desde el inicio en la máquina que alojará el gateway en la fábrica de San Pablo. No la generes en la computadora del desarrollador ni la envíes por mensajería. Se necesita OpenSSL; reemplazar los valores de ejemplo y usar la CUIT que WSASS requiere para la persona conectada:

```sh
cd /opt/arca-gateway/repo/deploy
mkdir -p secrets && chmod 700 secrets
openssl genrsa -out secrets/arca_private_key.pem 2048
openssl req -new -key secrets/arca_private_key.pem -subj "/C=AR/O=NombreDelNegocio/CN=ModuloFacturacion/serialNumber=CUIT 20123456789" -out secrets/arca_homologacion.csr
```

En `serialNumber`, escribir `CUIT`, un espacio y los 11 dígitos sin guiones. En el ejemplo, reemplazar `20123456789` por la CUIT que corresponda a la persona conectada a WSASS, según las validaciones de ese servicio.

**Guía oficial para generar el CSR:** [WSASS: cómo generar una solicitud de certificado](https://www.arca.gob.ar/ws/WSASS/html/generarcsr.html).

**La clave privada es `secrets/arca_private_key.pem`.** No se pega en WSASS, no se sube a Flutter y no se envía por correo, WhatsApp ni este chat. El archivo público que se carga en WSASS es `secrets/arca_homologacion.csr`.

### C. Obtener el certificado en WSASS

1. En WSASS, abrir **Nuevo Certificado**.
2. Elegir un alias nuevo y pegar el contenido completo de `secrets/arca_homologacion.csr`, incluidas las líneas `BEGIN CERTIFICATE REQUEST` y `END CERTIFICATE REQUEST`.
3. Crear el certificado y guardarlo directamente en la máquina gateway como `deploy/secrets/arca_certificate.pem`. Ese archivo es el certificado público; debe corresponder con la clave privada generada junto al CSR.

Si WSASS rechaza el CSR, verificar que se haya pegado el CSR y no la clave privada, que el alias no exista y que la CUIT del CSR coincida con la requerida por la sesión de WSASS.

**Guía oficial para crear el certificado:** [WSASS: cómo crear un certificado nuevo](https://www.arca.gob.ar/ws/WSASS/html/crearcertificado.html).

### D. Autorizar el certificado para el emisor y WSFEv1

1. En WSASS, abrir **Crear autorización a servicio**.
2. Seleccionar como **CUIT representada** la CUIT emisora del negocio.
3. Seleccionar el servicio de factura electrónica WSFEv1 (`wsfe` o el nombre equivalente que muestre el catálogo).
4. Confirmar y verificar que la autorización quede registrada.

Generar el certificado no alcanza por sí solo: también tiene que existir esta autorización para la CUIT emisora. ARCA explica que en homologación se gestiona en WSASS y que los certificados de testing no son válidos en producción.

**Guía oficial de autorización:** [WSASS: cómo crear una autorización de acceso](https://www.arca.gob.ar/ws/WSASS/html/crearautorizacion.html). Se puede revisar el nombre exacto del servicio en el [catálogo de WSASS](https://www.arca.gob.ar/ws/WSASS/html/verservicios.html).

### E. Guardarlo para el backend

El certificado y la clave viven **solo en el gateway de la máquina fija de la empresa** (V2). La clave debe generarse y quedarse en un equipo bajo control de la empresa; no la envíes al desarrollador. Desde `gateway/`, revisa el par sin contactar a ARCA:

```sh
npm run arca:check -- --cert ../deploy/secrets/arca_certificate.pem --key ../deploy/secrets/arca_private_key.pem --env homologation --cuit <CUIT emisora>
```

Si no hay `[ERROR]`, conserva ambos archivos en `deploy/secrets/` de la máquina gateway y sigue [deploy/INSTALACION.md](deploy/INSTALACION.md). En Firebase solo se carga el secreto compartido para firmar las solicitudes (`ARCA_GATEWAY_HMAC_SECRET`).

Configuración en `functions/.env.<id-del-proyecto>`:

- `ARCA_ENVIRONMENT=homologation`
- `ARCA_GATEWAY_URL=https://<dominio del gateway>`
- `ARCA_ISSUER_CUIT=<CUIT emisora, 11 dígitos, igual que en el gateway>`
- `ARCA_ISSUER_TAX_CONDITION=<responsable_inscripto | monotributo | exento>`
- `ARCA_POINT_OF_SALE=<punto habilitado para Web Services>`
- `DNI_THRESHOLD_AMOUNT` vacío = $10.000.000 (RG 5700/2025)

Después de desplegar al proyecto clonado con `firebase deploy --project <ID_DEL_CLON> --only functions:default,functions:arca`, un usuario `billingAdmin` ejecuta el diagnóstico `arcaHealthCheck` desde la app (`ArcaInvoiceClient().healthCheck()`): prueba el gateway, FEDummy, el inicio de sesión WSAA, la autorización `wsfe`, el punto de venta y el último número autorizado, sin emitir comprobantes.

No cargar certificado/clave en Flutter, Firestore, Firebase ni variables públicas de la app. No compartir la clave privada por mensajería. Si no se genera directamente en el gateway, transferirla desde un equipo controlado por la empresa por un canal cifrado directo y eliminar copias temporales; mantener un respaldo cifrado bajo custodia del titular.

## 3. Reglas fiscales y datos de cada pedido

Esto no se resuelve solo con una respuesta técnica de ARCA. El contador debe confirmar cómo se trata fiscalmente la venta del negocio y revisar el mapeo antes de emitir.

### Tipo de factura

No decidir A o B solamente por un selector del tipo de cliente. La clase depende de la condición fiscal del **emisor y del receptor**, y de las autorizaciones que tenga el emisor. Como orientación general para operaciones locales, ARCA publica que un responsable inscripto suele emitir B a consumidor final/exento y A a responsables inscriptos/monotributistas; un emisor monotributista o exento suele emitir C. Hay excepciones y variantes de comprobantes A, por lo que el contador debe confirmar la regla aplicable y qué tipos están autorizados para este CUIT.

El módulo aplica hoy esta regla según `ARCA_ISSUER_TAX_CONDITION`: emisor RI → Factura A a RI y monotributistas, Factura B a consumidor final y exentos; emisor monotributista o exento → Factura C a todos. El contador debe confirmarla para este CUIT.

### Información que debe aportar el negocio/contador para mapear el pedido

- Condición fiscal del emisor y comprobantes autorizados (A, B, C u otros que correspondan).
- Tipo de documento y número del receptor; condición frente al IVA del receptor cuando corresponda. La condición se valida con la documentación fiscal del cliente o por el procedimiento que determine el contador.
- Concepto fiscal de la operación. Para venta de colchones como productos, normalmente se analiza como venta de bienes; confirmar el código con el contador.
- Importes del pedido separados por tratamiento fiscal: neto gravado por cada alícuota, IVA por alícuota, no gravado, exento y otros tributos, si corresponden.
- Moneda y cotización. La versión actual del adaptador se limita a ARS.
- Si se agregan servicios u operaciones mixtas: fechas de servicio y vencimiento que requiere WSFEv1.
- Regla vigente para identificar al consumidor final cuando se requiera documento. No fijar en la app un umbral legal sin confirmación del contador.

Los códigos de condición IVA, alícuotas, tipos de documento y comprobante se verifican contra los parámetros/tablas de WSFEv1, por ejemplo `FEParamGetCondicionIvaReceptor`, `FEParamGetTiposIva`, `FEParamGetTiposCbte` y `FEParamGetTiposDoc`. El módulo actual no calcula automáticamente el impuesto a partir del precio del producto: espera recibir el resumen fiscal del pedido en `invoiceFiscalData`.

**Lecturas oficiales:** [clases de comprobantes según emisor y receptor](https://www.arca.gob.ar/facturacion/regimen-general/comprobantes.asp) y [manuales y tablas de WSFEv1](https://arca.gob.ar/ws/documentacion/ws-factura-electronica.asp).

## 4. Qué pedirle a cada persona

### Al titular o contador

- CUIT emisora y constancia de inscripción vigente.
- Condición fiscal del emisor y tipos de factura/clases habilitados.
- Número de punto de venta existente para Web Services, o confirmación de que deben dar de alta uno.
- Aprobación del mapeo de tipo de comprobante y condición IVA del comprador.
- Regla de documento para consumidor final y validación de importes/alícuotas.

### A quien administre ARCA/WSASS

- Confirmación de acceso a WSASS con CUIT personal.
- CSR cargado, certificado PEM descargado y autorización para la CUIT emisora y servicio WSFEv1.
- Generar la clave privada bajo control de la empresa y conservarla solo en el gateway y en un respaldo cifrado custodiado por el titular.

### Al desarrollador de la aplicación

- Acceso al código fuente y al backend/proyecto donde están los pedidos (Firebase u otro).
- Estructura real del pedido: colección/API, campos de precio, cliente, documento, impuestos y estado de entrega.
- Acceso de despliegue al proyecto donde correrá la función y forma de autenticar las llamadas desde la app.

Firebase y el acceso al código no los entrega ARCA. Si la app ya tiene backend, el módulo se integra allí; si no, se despliega el backend pequeño de facturación y se acuerda cómo leer y actualizar pedidos de forma segura.

## 5. Alcance y seguridad

- Primero probar solo en homologación. Sus comprobantes no habilitan la entrega como factura fiscal.
- No usar el certificado de homologación para producción. Producción requiere clave, CSR, certificado y autorización nuevos: el titular, con Clave Fiscal nivel 3, crea el certificado en **Administración de Certificados Digitales** y lo asocia a **Facturación Electrónica (wsfe)** en **Administrador de Relaciones de Clave Fiscal**. El servicio ya soporta producción con `ARCA_ENVIRONMENT=production` (ver la sección "Producción" de [HOMOLOGACION.md](HOMOLOGACION.md)).
- No compartir Clave Fiscal ni clave privada con el equipo técnico por chats. El titular debe realizar personalmente los pasos fiscales y cargar los secretos en el entorno seguro acordado.
- Si WSFEv1 devuelve un CAE pero la conexión se corta antes de que el sistema reciba la respuesta, no emitir de nuevo a ciegas: hay que consultar/reconciliar el comprobante para evitar duplicarlo.

## Referencias oficiales

- [Guía ARCA: acciones para consumir un Web Service de factura electrónica](https://www.arca.gob.ar/fe/documentos/AccionesarealizarparaconsumirunWebservicedeFacturaElectr.pdf)
- [Entorno de prueba y ayuda para homologación](https://arca.gob.ar/fe/ayuda/entorno-prueba.asp)
- [Certificados para web services y acceso a WSASS](https://www.arca.gob.ar/ws/documentacion/certificados.asp)
- [Manual completo de WSASS (PDF)](https://www.arca.gob.ar/ws/WSASS/WSASS_manual.pdf)
- [Manual de usuario WSASS: conceptos y autorizaciones](https://www.arca.gob.ar/ws/WSASS/html/conceptos.html)
- [WSASS: cómo crear un certificado](https://www.arca.gob.ar/ws/WSASS/html/crearcertificado.html)
- [WSASS: cómo generar un CSR](https://www.arca.gob.ar/ws/WSASS/html/generarcsr.html)
- [ARCA: documentación de WSAA](https://www.arca.gob.ar/ws/documentacion/wsaa.asp)
- [ARCA: documentación WSFEv1 y manuales](https://arca.gob.ar/ws/documentacion/ws-factura-electronica.asp)
- [ARCA: arquitectura de los Web Services y autenticación WSAA](https://arca.gob.ar/ws/documentacion/arquitectura-general.asp)
- [ARCA: puntos de venta y solicitud de autorización](https://arca.gob.ar/fe/emision-autorizacion/solicitud-autorizacion.asp)
- [ARCA: clases de comprobantes según emisor y receptor](https://www.arca.gob.ar/facturacion/regimen-general/comprobantes.asp)
- [ARCA: constancia de inscripción](https://www.arca.gob.ar/buscador-consultas-publicas/)
