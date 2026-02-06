# Webhooks de Yavu hacia Vandalia

Documento de referencia para el equipo de Yavu sobre cómo enviar eventos de comprobantes a la plataforma de Vandalia a través de webhooks.

## 1. Endpoint

- **Método**: `POST`
- **Content-Type**: `application/json`

| Entorno    | URL de webhook                                     |
|-----------|-----------------------------------------------------|
| Producción | `https://app.vandalia.com.ar/api/webhooks/yavu/`   |


---

## 2. Autenticación y seguridad

### 2.1. Canal seguro (HTTPS)

- Todos los requests deben realizarse sobre **HTTPS**.
- No se acepta tráfico sin cifrar.

### 2.2. Firma de seguridad (recomendada)

Para asegurar que el webhook proviene efectivamente de Yavu, se usará una **clave secreta compartida** (string aleatoria) que **no aparece en este documento** y se entrega por canal privado.

**Header propuesto:**

- `X-Yavu-Signature`: firma HMAC-SHA256 del cuerpo crudo (`raw body`) en formato **hexadecimal**.

**Algoritmo (lado Yavu):**

1. Tomar el cuerpo del request exactamente como se envía (JSON en bytes).
2. Calcular:  
   `HMAC_SHA256(clave_secreta, raw_body)`  
3. Convertir el resultado a **hex** (ej. `b1a4e3...`).
4. Enviar ese valor en el header `X-Yavu-Signature`.

Del lado de Vandalia, se recalcula la firma con la misma clave y se compara:

- Si la firma **coincide** → el request se considera auténtico.
- Si la firma **no coincide** o **no existe** → el request puede rechazarse con **401 Unauthorized**.

> La **clave secreta** y, si aplica, el valor exacto del header se compartirá con Yavu por un canal privado (no en esta documentación).

---

## 3. Formato del request

### 3.1. Campos obligatorios

El body debe ser un JSON con al menos estos campos:

| Campo        | Tipo    | Descripción                                                                 |
|--------------|---------|-----------------------------------------------------------------------------|
| `event`      | string  | Identificador del evento (ver tabla de eventos soportados más abajo).      |
| `referencia` | string  | Identificador unívoco del comprobante. Debe ser el mismo valor enviado por Vandalia a Yavu en `grabaComprobante`. |

Cualquier otro campo que se envíe se almacena para trazabilidad y diagnóstico (por ejemplo: `empresa`, `comprobante`, `talon`, `numero`, etc.).

### 3.2. Ejemplos de request

**Ejemplo mínimo:**

```json
{
  "event": "comprobante.packed_despacho",
  "referencia": "1884818991"
}
```

**Ejemplo con datos adicionales:**

```json
{
  "event": "comprobante.facturado",
  "referencia": "1884818991",
  "empresa": 2,
  "comprobante": 4,
  "talon": 40,
  "numero": 17
}
```

---

## 4. Respuestas del webhook

### 4.1. Códigos HTTP

- **200 OK**  
  - El JSON es válido y contiene `event` y `referencia`.
  - El evento fue **recibido** y se intentó procesar internamente.
  - Errores de negocio (orden no encontrada, imposibilidad de aplicar la acción) no cambian el código HTTP (se informan en el cuerpo).

- **400 Bad Request**  
  - JSON mal formado, o falta alguno de los campos obligatorios (`event`, `referencia`).

- **401 Unauthorized** (si se activa la firma de seguridad)  
  - Firma ausente o inválida en el header `X-Yavu-Signature`.

- **5xx**  
  - Error interno no esperado. En este caso, Yavu puede **reintentar** el envío más adelante.

### 4.2. Cuerpo de la respuesta (200 OK)

Respuesta básica cuando se recibe y valida el request:

```json
{
  "received": true,
  "event": "comprobante.packed_despacho",
  "referencia": "1884818991"
}
```

Cuando existe una orden asociada a la referencia:

- `"order_found": true`
- `"processed": true` → la acción asociada al evento se aplicó correctamente.  
- `"processed": false` → hubo un error aplicando la acción; el error queda registrado internamente para revisión.

Ejemplo:

```json
{
  "received": true,
  "event": "comprobante.facturado",
  "referencia": "1884818991",
  "order_found": true,
  "processed": true
}
```

Cuando **no** existe integración/orden para la referencia enviada:

```json
{
  "received": true,
  "event": "comprobante.packed_despacho",
  "referencia": "1884818991",
  "order_found": false
}
```

---

## 5. Eventos soportados

La siguiente tabla describe los valores esperados en `event` y el significado de cada uno en el flujo de la orden:

| `event`                           | Descripción funcional                                            |
|-----------------------------------|------------------------------------------------------------------|
| `comprobante.packed_despacho`    | El comprobante fue empaquetado para despacho.                   |
| `comprobante.packed_retiro`      | El comprobante fue empaquetado para retiro en sucursal.         |
| `comprobante.packed`             | Empaquetado genérico (cuando no se distingue despacho/retiro).  |
| `comprobante.facturado`          | El comprobante fue facturado.                                   |
| `comprobante.cancelled_stock`    | El comprobante se canceló por quiebre de stock.                 |
| `comprobante.cancelled`          | El comprobante se canceló por otro motivo.                      |
| `comprobante.delivered_sucursal` | El comprobante se entregó en sucursal.                          |

- Cualquier otro valor en `event`:
  - Se registra para traza e historial de la orden.
  - Si no hay lógica específica asociada a ese evento, no se realizan cambios adicionales sobre la orden.

---

## 6. Relación con la referencia (`referencia`)

- La orden se identifica siempre por el campo `referencia`.
- Este valor debe coincidir exactamente con el que envía Vandalia a Yavu al crear el comprobante (campo `referencia` en `grabaComprobante`).
- Para que el flujo sea consistente:
  - Yavu debe **almacenar** el valor de `referencia` recibido al momento de grabar el comprobante.
  - Cualquier webhook posterior sobre ese comprobante debe reenviar **el mismo valor** de `referencia`.

Ejemplo del fragmento enviado desde Vandalia al crear el comprobante:

```json
{
  "empresa": 2,
  "tipoComprobante": 4,
  "talon": 40,
  "numero": 0,
  "referencia": "1884818991",
  "notas": "Orden #1884818991"
}
```

En todos los webhooks relacionados con ese comprobante, `referencia` debe ser `"1884818991"`.

---

## 7. Idempotencia y reintentos

- El receptor de los webhooks está preparado para recibir el **mismo evento varias veces** para la misma `referencia`.
- Cada request se registra con su payload completo para trazabilidad.
- Yavu **no necesita** hacer deduplicación estricta para esta primera versión: si un evento se reenvía, el sistema:
  - Vuelve a registrar el evento.
  - Aplica la lógica de negocio de forma idempotente (sin duplicar efectos).

**Recomendación de reintentos:**

- Reintentar el webhook cuando:
  - Se reciba un **5xx**.
  - Haya un timeout de red.
- **No** es necesario reintentar cuando:
  - La respuesta sea **200 OK** (aunque `processed` sea `false`; el error ya quedó registrado internamente).
  - La respuesta sea **400** o **401**: en esos casos hay un problema de payload o autenticación que debe corregirse antes de reintentar.

---

## 8. Checklist para implementación en Yavu

1. **Al grabar el comprobante**  
   - Guardar el valor de `referencia` recibido desde Vandalia.

2. **Eventos que disparan webhooks**  
   - Empaquetado para despacho → enviar `comprobante.packed_despacho`.  
   - Empaquetado para retiro → enviar `comprobante.packed_retiro`.  
   - Facturado → enviar `comprobante.facturado`.  
   - Cancelado por quiebre de stock → enviar `comprobante.cancelled_stock`.  
   - Cancelado por otro motivo → enviar `comprobante.cancelled`.  
   - Entregado en sucursal → enviar `comprobante.delivered_sucursal`.

3. **Formato del POST**  
   - `Content-Type: application/json`.
   - Body JSON con `event` y `referencia` obligatorios.
   - Opcionalmente: `empresa`, `comprobante`, `talon`, `numero`, y cualquier otro dato útil para trazabilidad.

4. **Seguridad**  
   - Enviar siempre por HTTPS.
   - Implementar la firma `X-Yavu-Signature` usando la clave compartida que se enviará por canal privado.

5. **Pruebas**  
   - Validar primero contra el endpoint de desarrollo (URL + clave compartida provista por Vandalia).
   - Verificar:
     - Código HTTP.
     - Campos `received`, `order_found`, `processed` en la respuesta.

