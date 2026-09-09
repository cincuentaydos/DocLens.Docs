# Alta de Tenants

Esta página describe cómo un nuevo tenant (despacho de abogados) obtiene acceso a DocLens: cómo se provisionan las credenciales, cómo se asigna el `tenantId`, y cómo se obtiene el token de autenticación para las llamadas a la API.

---

## Modelo de Identidad

DocLens usa **Amazon Cognito** como proveedor de identidad. Cada solicitud de API lleva un JWT emitido por Cognito (ID token) en el encabezado `Authorization`. El token contiene un atributo personalizado — `custom:tenantId` — que vincula al usuario autenticado con su tenant (despacho).

El middleware de tenant en la Lambda lee este claim en cada solicitud. Todas las operaciones downstream (claves S3, filas en Aurora, filtros de Bedrock, campos de log) quedan acotadas a ese `tenantId`. No hay forma de actuar en nombre de un tenant distinto dentro de una misma solicitud.

---

## Tipos de Usuario

Dentro de un mismo tenant existen dos categorías de usuario (ver `architecture.md` → Modelo de Multi-Tenancy):

| Tipo | Descripción | Alcance |
|---|---|---|
| `interno_admin` | Usuario interno del despacho con privilegios de administración (crear usuarios, clientes, otorgar permisos, consultar auditoría) | Todo el tenant |
| `interno` | Usuario interno del despacho sin privilegios de administración | Los procesos para los que tiene permiso explícito (ver `api-reference.md` → Permisos) |
| `cliente` | Usuario del cliente del despacho — consulta el estado y los documentos de sus propios procesos | Solo los procesos asociados a su `clienteId` |

El tipo de usuario y, cuando aplica, el `clienteId`, se almacenan como atributos personalizados adicionales en Cognito (`custom:tipoUsuario`, `custom:clienteId`), leídos por el middleware de tenant junto con `custom:tenantId`.

Todo acceso de un usuario a un proceso o documento queda registrado en la tabla `audit_log` (ver [ADR-008](adrs/008-data-storage-strategy.md)) y es consultable vía `GET /audit-log` (ver `api-reference.md`) — parte del requisito de auditar el control de accesos.

---

## Convención de Tenant ID

Un `tenantId` es un **slug alfanumérico en minúsculas** asignado en el momento del alta y que nunca cambia.

```
despacho-acme
despacho-globex
despacho-initech
```

El slug se usa como componente de ruta en las claves S3 y como valor de la columna `empresa_id` en Aurora, por lo que debe:

- Contener solo letras minúsculas, dígitos y guiones.
- Empezar con una letra.
- Ser único entre todos los tenants.
- Ser estable — cambiarlo dejaría huérfanos todos los documentos existentes.

---

## Configuración del User Pool de Cognito

DocLens usa un **único User Pool de Cognito** compartido entre todos los tenants. El aislamiento por tenant se aplica vía el atributo `custom:tenantId`, no mediante pools separados.

| Configuración | Valor |
|---|---|
| User Pool | Uno por entorno de DocLens (dev / staging / production) |
| App Client | Uno por entorno, usado por el frontend |
| Flujos de auth | `USER_PASSWORD_AUTH` (dev); `USER_SRP_AUTH` (staging / production) |
| Atributo personalizado | `custom:tenantId` — de solo lectura tras asignarse; no modificable por el usuario |
| Token | ID token (lleva los atributos personalizados); el access token no |
| TTL del token | 1 hora (ID token); 30 días (refresh token — configurable) |

!!! warning "ID token, no access token"
    La Lambda valida el **ID token** (`Authorization: Bearer <id-token>`). El access token no lleva atributos personalizados y fallará la verificación del claim `tenantId`.

---

## Flujo de Alta

El alta de un tenant es un **proceso gestionado por un administrador** — los tenants no se autorregistran. Un nuevo tenant se provisiona mediante la AWS CLI o Terraform.

### Paso 1 — Asignar un tenant ID

Elegir un slug único siguiendo la convención anterior. Este es el identificador canónico — registrarlo en el registro de tenants.

### Paso 2 — Crear el primer usuario interno admin

```bash
aws cognito-idp admin-create-user \
  --user-pool-id <pool-id> \
  --username <email> \
  --user-attributes \
      Name=email,Value=<email> \
      Name=email_verified,Value=true \
      Name=custom:tenantId,Value=<tenant-slug> \
      Name=custom:tipoUsuario,Value=interno_admin \
  --temporary-password <temp-password> \
  --message-action SUPPRESS
```

El atributo `custom:tenantId` se fija aquí y no puede ser cambiado por el usuario — Cognito lo impone vía la configuración `Mutable: false` del pool. Usuarios internos adicionales y usuarios de cliente se crean posteriormente vía `POST /usuarios` y `POST /clientes/{clienteId}/usuarios` respectivamente (ver `api-reference.md`), no directamente por CLI.

### Paso 3 — Forzar el cambio de contraseña

En el primer inicio de sesión, Cognito exige que el usuario establezca una contraseña permanente. Esto ocurre vía el flujo de login del frontend o directamente vía CLI:

```bash
aws cognito-idp admin-set-user-password \
  --user-pool-id <pool-id> \
  --username <email> \
  --password <permanent-password> \
  --permanent
```

### Paso 4 — Entregar credenciales

Proveer al tenant:

- Su correo de login.
- Su contraseña inicial (o permanente).
- La URL del frontend para su entorno.

---

## Flujo de Autenticación (Cliente)

Una vez provisionado, un usuario se autentica vía la UI alojada de Cognito o directamente usando la API de Cognito:

```
POST https://cognito-idp.eu-west-1.amazonaws.com/
X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth

{
  "AuthFlow": "USER_PASSWORD_AUTH",
  "ClientId": "<app-client-id>",
  "AuthParameters": {
    "USERNAME": "<email>",
    "PASSWORD": "<password>"
  }
}
```

**Respuesta:**

```json
{
  "AuthenticationResult": {
    "IdToken": "eyJ...",
    "AccessToken": "eyJ...",
    "RefreshToken": "eyJ...",
    "ExpiresIn": 3600
  }
}
```

El `IdToken` es el token bearer usado en todas las llamadas a la API de DocLens:

```
Authorization: Bearer <IdToken>
```

---

## Renovación de Tokens

Los tokens expiran después de 1 hora. El frontend los renueva automáticamente usando el `RefreshToken`:

```
POST https://cognito-idp.eu-west-1.amazonaws.com/
X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth

{
  "AuthFlow": "REFRESH_TOKEN_AUTH",
  "ClientId": "<app-client-id>",
  "AuthParameters": {
    "REFRESH_TOKEN": "<refresh-token>"
  }
}
```

---

## Baja de un Tenant

Para revocar el acceso de un tenant:

1. **Deshabilitar todos los usuarios de Cognito** de ese tenant — impide la emisión de nuevos tokens. Los tokens existentes siguen siendo válidos hasta su expiración (máximo 1 hora).
2. **Opcionalmente eliminar sus datos** de S3 y Aurora usando el `tenantId`/`empresa_id` como filtro.
3. **Opcionalmente eliminar los datos de su base de conocimiento** disparando un job de eliminación de ingesta acotado a su filtro de metadatos `empresa_id`.

!!! danger "La eliminación de datos es irreversible"
    Eliminar objetos de S3 y filas de Aurora no puede deshacerse. Confirmar siempre el `tenantId` antes de ejecutar eliminaciones masivas.

---

## Preguntas Abiertas

- ¿Debería automatizarse el alta de tenants vía un endpoint de gestión en lugar de comandos CLI de administrador?
- ¿Cuál es la política de retención de datos tras la baja — eliminación inmediata o un período de gracia?
