# GitHub Copilot: Usage Report por API y evidencia de custom agents (Windows)

Guia de demostracion. Fuentes consultadas el 8 de octubre de 2026.

En Windows, los apartados de Usage Report usan **Git Bash** porque incluyen
sintaxis Bash, `curl` y `jq`. La demostracion OpenTelemetry del apartado 4
usa **PowerShell** y no requiere `jq`.

## 1. Que esta comprobado

| Afirmacion | Evidencia y alcance |
| --- | --- |
| Existe una API oficial de exportacion CSV. | La documentacion admite `ai_credit`. Se genero y descargo un informe real de septiembre en una enterprise de pruebas. |
| El CSV permite consumir datos de creditos, importes y tokens. | El informe real tuvo 69 filas. Cantidad, bruto, descuento y neto coincidieron exactamente con la API JSON en 15 grupos: 60 comparaciones. |
| El nombre del custom agent puede recuperarse de la telemetria. | Dos ejecuciones completas con VS Code 1.140.0 y Copilot Chat 0.68.0: el cliente emitio `copilot_chat.mode_name` en spans `invoke_agent`. |

**Modelo de esta demostracion:** `gpt-5.6-luna`, servido por GitHub Copilot.
El apartado 4 explica como seleccionarlo y comprobarlo en la telemetria.
Docker ejecuta solamente el Collector, no el modelo.

## 2. Tipos de `report_type`

El endpoint admite estos cuatro valores exactos:

| Valor | Informe |
| --- | --- |
| `summarized` | Uso facturable agregado de productos. |
| `detailed` | Uso facturable detallado; incorpora campos como usuario y workflow cuando aplican. |
| `premium_request` | Consumo de solicitudes premium. No es el informe de AI credits. |
| `ai_credit` | Consumo de AI credits por fecha, modelo y usuario, con importes y tokens de entrada, salida y cache. **Es el utilizado en esta evaluacion.** |

La referencia de billing documenta hasta un ano para el resumido y hasta
31 dias para el detallado y el de AI credits. El rango de septiembre del
ejemplo tiene 30 dias. Las fechas de uso son UTC.

**API de exportacion, no `/usage`:**

```text
POST https://api.github.com/enterprises/{enterprise}/settings/billing/reports
GET  https://api.github.com/enterprises/{enterprise}/settings/billing/reports/{report_id}
GET  https://api.github.com/enterprises/{enterprise}/settings/billing/reports
```

El aviso de la referencia de billing sobre que `/usage` no devuelve todo el
detalle del CSV NO significa que no exista la API de exportacion `/reports`.

## 3. cURL para una demo de septiembre

Estos ejemplos estan escritos para **Bash**, con `curl` y `jq`. Ejecutarlos
crea archivos locales; el POST crea un informe en la enterprise elegida.
No reintentar el POST automaticamente: reutilizar el `id` devuelto.

Requisitos:

- Un token autorizado en la variable de entorno `GITHUB_BILLING_TOKEN`.
  Debe incluir el permiso (scope) *`manage_billing:enterprise`*.
- El slug de una enterprise, no el de una organizacion.
- Una carpeta nueva para evitar sobrescribir resultados.

```bash
umask 077
mkdir demo-api-septiembre
cd demo-api-septiembre
export ENTERPRISE="SLUG-DE-TU-ENTERPRISE"
export API="https://api.github.com/enterprises/${ENTERPRISE}/settings/billing/reports"
: "${GITHUB_BILLING_TOKEN:?La variable GITHUB_BILLING_TOKEN no esta definida}"
```

### A. Crear el informe de AI credits

```bash
curl --silent --show-error --fail-with-body \
  --request POST "$API" \
  --header "Accept: application/vnd.github+json" \
  --header "Authorization: Bearer ${GITHUB_BILLING_TOKEN}" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header "Content-Type: application/json" \
  --data '{
    "report_type": "ai_credit",
    "start_date": "2026-09-01",
    "end_date": "2026-09-30",
    "send_email": false
  }' \
  --output creado.json \
  --write-out 'HTTP %{http_code}\n'

jq '{id, report_type, start_date, end_date, status}' creado.json
export REPORT_ID="$(jq -er '.id' creado.json)"
```

Resultado esperado: **HTTP 202**, un `id` y normalmente `status: "processing"`.
El tiempo de unos 86 segundos observado en la evaluacion no es un SLA.

### B. Consultar el estado

```bash
curl --silent --show-error --fail-with-body \
  "$API/$REPORT_ID" \
  --header "Accept: application/vnd.github+json" \
  --header "Authorization: Bearer ${GITHUB_BILLING_TOKEN}" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --output estado.json

jq '{id, status, partes: ((.download_urls // []) | length)}' estado.json
```

Si sigue en `processing`, esperar y repetir **este GET**, no el POST.
Si devuelve `failed`, detener la demo y mostrar el error.
Una automatizacion debe tener un tiempo maximo y tratar errores HTTP,
incluidos permisos, limites y fallos transitorios. `billing_export.py` ya
implementa esas comprobaciones.

### C. Descargar todas las partes cuando termine

Las URLs firmadas conceden acceso al fichero. No proyectarlas ni compartirlas.
La descarga **no lleva el token de GitHub**.

```bash
if jq -e '.status == "completed" and
  (.download_urls | type == "array" and length > 0)' estado.json >/dev/null
then
  parte=0
  while IFS= read -r url; do
    parte=$((parte + 1))
    archivo="$(printf 'ai-credit-septiembre-%03d.csv' "$parte")"
    if [ -e "$archivo" ] || [ -e "$archivo.part" ]; then
      printf 'No se sobrescribe %s; usa otra carpeta.\n' "$archivo" >&2
      break
    fi
    if curl --silent --show-error --fail --proto '=https' \
      --output "$archivo.part" "$url"
    then
      mv "$archivo.part" "$archivo"
      printf 'Descargado: %s\n' "$archivo"
    else
      printf 'Descarga fallida; %s.part NO es un informe completo.\n' "$archivo" >&2
      break
    fi
  done < <(jq -er '.download_urls[]' estado.json)
else
  printf 'Informe no completado o sin URLs de descarga.\n' >&2
fi
```

Es una secuencia didactica, no un job de produccion. El cliente Python incluido
valida tambien identidad y fechas del informe, esquema CSV, hosts de descarga
y todas las partes antes de publicar una salida completa.

### D. Listar informes ya solicitados

```bash
curl --silent --show-error --fail-with-body "$API" \
  --header "Accept: application/vnd.github+json" \
  --header "Authorization: Bearer ${GITHUB_BILLING_TOKEN}" \
  --header "X-GitHub-Api-Version: 2026-03-10" |
  jq '.usage_report_exports[] |
    {id, report_type, start_date, end_date, status}'
```

Permite elegir un informe previo y continuar desde el GET, sin otro POST.
Los registros completados o fallidos se conservan durante 31 dias. Esa
retencion no es el limite del rango de fechas del informe.

### Permisos, correo y programacion

El endpoint admite usuarios autorizados como administradores de enterprise,
billing managers o roles personalizados con lectura de facturacion.
El token almacenado en `GITHUB_BILLING_TOKEN` debe incluir el permiso
(scope) *`manage_billing:enterprise`*.
Tambien admite el installation access token de una **GitHub App instalada
en la enterprise**, con **Enterprise billing: Read**. Una instalacion
solo en una organizacion no sustituye a la instalacion empresarial.
La modalidad GitHub App esta documentada; no se creo una App durante el lab.
El [anexo A](#anexo-a-usage-report-con-github-app-comandos-curl) contiene
la secuencia completa de autenticacion y los comandos cURL.

`send_email: true` notifica al solicitante. No define frecuencia ni destinatario
arbitrario. La recurrencia y el envio a un buzon compartido deben resolverse
con una automatizacion corporativa autorizada; no se desplegaron en este lab.

## 4. Demostracion manual con GitHub Copilot y `gpt-5.6-luna`

**No confundir las dos demostraciones:** Usage Report es una llamada a GitHub
y no necesita Docker ni VS Code. En esta segunda demostracion, VS Code
GENERA telemetria y Docker solo la RECIBE, filtra y guarda.

```text
VS Code / Copilot -> HTTPS -> GitHub Copilot: gpt-5.6-luna
       |
       +-> OTLP HTTP local -> Collector Docker
                                | received.jsonl: antes
                                | sanitized.jsonl: atributos necesarios
```

Esta demostracion utiliza la autenticacion normal de VS Code y el modelo
**`gpt-5.6-luna` de GitHub Copilot**, no un proveedor local ni una clave de
OpenAI. Requiere una cuenta con Copilot y acceso a ese modelo, conexion a
GitHub y Docker local. **Puede consumir creditos de Copilot.**

No cargar tokens de billing, credenciales ficticias ni overrides de endpoints.
Usar un equipo/laboratorio autorizado, sin modificar politicas corporativas.
Si el modelo no esta disponible, detener esta variante; no sustituirlo
silenciosamente por Auto, otro modelo o `local-fixture`.

### Paso 1. Preparar una carpeta de pruebas

Descomprimir el ZIP de cliente. Abrir PowerShell en esa carpeta.
Crear una copia minima, sin reutilizar las capturas existentes.

En PowerShell:

```powershell
New-Item -ItemType Directory -Path demo-manual
Copy-Item compose.yaml, collector.yaml -Destination demo-manual
Copy-Item workspace -Destination demo-manual -Recurse
Set-Location demo-manual
New-Item -ItemType Directory -Path output
```

`workspace/.github/agents/` ya contiene Lab Planner, Lab Reviewer y
Lab Unmapped. Son agentes de prueba sin herramientas; no se abre ningun
repositorio del cliente. Por ejemplo, planner.agent.md contiene:

```markdown
---
name: Lab Planner
description: Synthetic telemetry test without tools.
tools: []
---
Reply only OK. Do not use tools.
```

Los nombres permiten comprobar directamente el valor emitido en
`copilot_chat.mode_name`; el Collector no les asigna ningun ID adicional.

### Paso 2. Arrancar el Collector

Con Docker Desktop ya disponible y configurado para contenedores Linux:

```powershell
$env:COMPOSE_PROJECT_NAME = "copilot-otel-demo-manual"
docker compose up -d
docker compose ps
docker compose port collector 4318
```

El ultimo comando muestra algo parecido a `127.0.0.1:54321`.
**Copiar el puerto real**, no asumir que sera 4318: Compose asigna uno libre
en el host. El contenedor escucha OTLP HTTP internamente en 4318.
Todavia no debe esperarse ninguna conversacion en los ficheros.

Si el contenedor no aparece en ejecucion, consultar `docker compose logs collector`
y resolverlo antes de continuar.

### Paso 3. Abrir solo la carpeta de prueba en un perfil aparte

En PowerShell, con `code` disponible en `PATH`:

```powershell
$ProfileDir = Join-Path $env:TEMP ("copilot-demo-manual-" + [guid]::NewGuid())
New-Item -ItemType Directory -Path $ProfileDir
Write-Host "Perfil de pruebas: $ProfileDir"
code `
  --user-data-dir $ProfileDir `
  --extensions-dir (Join-Path $ProfileDir "extensions") `
  --new-window (Join-Path (Get-Location) "workspace")
```

Si `code` no esta en `PATH`, ejecutar el mismo comando con la ruta a
`code.cmd`, normalmente
`$env:LOCALAPPDATA\Programs\Microsoft VS Code\bin\code.cmd`.

La ventana tiene un perfil separado. Completar manualmente el inicio de sesion
de Copilot con una cuenta autorizada, sin pegar credenciales en el chat.
Si VS Code lo solicita, habilitar GitHub Copilot Chat. Comprobar las versiones
en Help > About y Extensions: la referencia historica es 1.140.0 / 0.68.0.
La cuenta debe tener acceso al modelo solicitado; ser enterprise owner
no sustituye a tener Copilot y ese modelo habilitado.
No hace falta cambiar el perfil habitual de trabajo.

**El aislamiento lo proporciona `--user-data-dir`.** Los cinco ajustes OTel
del paso siguiente tienen ambito `application` en Copilot Chat 0.68.0.
No ponerlos en `workspace/.vscode/settings.json`: no son ajustes de carpeta.
Tampoco sustituir este lanzamiento por cambiar simplemente de perfil en una
ventana habitual, porque eso no aisla los ajustes de ambito aplicacion.

### Paso 4. Configurar la salida OTel de ESA ventana

Abrir la paleta de comandos y elegir **Preferences: Open User Settings (JSON)**.
Anadir estas propiedades, usando el puerto del paso 2:

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.otlpEndpoint": "http://127.0.0.1:54321",
  "github.copilot.chat.otel.captureContent": false,
  "github.copilot.chat.otel.captureIdentity": false
}
```

Si ya hay propiedades, integrarlas en el objeto existente; no duplicar las
llaves exteriores. Guardar el archivo y ejecutar **Developer: Reload Window**.

No activar captureContent para esta comprobacion: no hace falta capturar
prompts para observar el nombre del modo. Variables OTel previas o ajustes
administrados pueden cambiar la configuracion efectiva; no eludir politicas.

### Paso 5. Seleccionar Lab Planner y `gpt-5.6-luna`

Abrir Chat y crear una sesion **Local de VS Code**. No elegir CLI,
background, cloud ni otro runtime: esta es la modalidad de la demostracion.
**Local se refiere a donde se ejecuta el agente; el modelo se ejecuta en
GitHub.** No significa que la inferencia sea local.

Elegir **Lab Planner** en el selector de agentes. En el selector de modelos,
elegir el modelo de GitHub Copilot cuyo identificador es **`gpt-5.6-luna`**.
No dejar **Auto** seleccionado. Si no aparece, comprobar la cuenta, el
catalogo disponible y las politicas con el administrador; no eludirlas.

Con el agente y el modelo seleccionados, enviar solamente:

> Responde solamente OK. No uses herramientas.

Esperar a que termine y dejar unos segundos para exportar el span.
Que el modelo diga "soy Lab Planner" NO es la prueba; lo son los atributos
recibidos en el Collector.

### Paso 6. Leer lo que VS Code ha emitido

En PowerShell no hace falta instalar `jq`. Todavia dentro de `demo-manual`,
definir primero esta funcion para convertir los atributos OTLP:

```powershell
function Convert-OtlpAttributes {
  param([object[]]$Attributes)
  $Result = [ordered]@{}
  foreach ($Attribute in $Attributes) {
    $ValueProperty = $Attribute.value.PSObject.Properties |
      Select-Object -First 1
    $Result[$Attribute.key] = $ValueProperty.Value
  }
  [pscustomobject]$Result
}
```

Despues leer `received.jsonl`:

```powershell
Get-Content output/received.jsonl | ForEach-Object {
  $Batch = $_ | ConvertFrom-Json
  foreach ($ResourceSpan in $Batch.resourceSpans) {
    $Resource = Convert-OtlpAttributes $ResourceSpan.resource.attributes
    if ($Resource.'service.name' -ne 'copilot-chat') { continue }
    foreach ($ScopeSpan in $ResourceSpan.scopeSpans) {
      foreach ($Span in $ScopeSpan.spans) {
        $Attributes = Convert-OtlpAttributes $Span.attributes
        if ($Attributes.'gen_ai.operation.name' -ne 'invoke_agent') { continue }
        [pscustomobject]@{
          spanId       = $Span.spanId
          traceId      = $Span.traceId
          status       = $Span.status
          agent        = $Attributes.'gen_ai.agent.name'
          mode         = $Attributes.'copilot_chat.mode_name'
          type         = $Attributes.'github.copilot.agent.type'
          model        = $Attributes.'gen_ai.request.model'
          conversation = $Attributes.'gen_ai.conversation.id'
        }
      }
    }
  }
} | Format-List
```

Buscar en un mismo span:

```text
gen_ai.agent.name         GitHub Copilot Chat
copilot_chat.mode_name    Lab Planner
github.copilot.agent.type custom
gen_ai.request.model     gpt-5.6-luna
```

Apuntar su `spanId`. El nombre procede directamente de VS Code y aparece en
`received.jsonl`; no lo genera el Collector.

Comprobar tambien que Copilot ha respondido y que la invocacion no tiene
estado de error. Un intento fallido con el nombre correcto del modelo no
demuestra que haya habido una respuesta real.

Si `model` es `local-fixture`, Auto u otro identificador, no dar esta prueba
por superada. **`gen_ai.provider.name=github` por si solo no basta**:
tambien aparecia en la prueba historica con modelo local.
Si se emite `gen_ai.response.model` en los spans de llamada al modelo,
conservarlo junto al modelo solicitado; puede identificar una version concreta.

### Paso 7. Comprobar la salida saneada

En la misma sesion de PowerShell, reutilizando `Convert-OtlpAttributes`
del paso anterior:

```powershell
Get-Content output/sanitized.jsonl | ForEach-Object {
  $Batch = $_ | ConvertFrom-Json
  foreach ($ResourceSpan in $Batch.resourceSpans) {
    foreach ($ScopeSpan in $ResourceSpan.scopeSpans) {
      foreach ($Span in $ScopeSpan.spans) {
        $Attributes = Convert-OtlpAttributes $Span.attributes
        [pscustomobject]@{
          spanId = $Span.spanId
          mode   = $Attributes.'copilot_chat.mode_name'
          type   = $Attributes.'github.copilot.agent.type'
          model  = $Attributes.'gen_ai.request.model'
        }
      }
    }
  }
} | Format-List
```

En el MISMO `spanId`, deben aparecer `mode: "Lab Planner"`,
`type: "custom"` y `model: "gpt-5.6-luna"`.

`copilot_chat.mode_name` es el dato emitido por el cliente que se utiliza
para identificar el custom agent en esta demostracion.

### Paso 8. Repetir con otro agente y otra conversacion

Sin abrir otra conversacion, seleccionar Lab Reviewer y enviar el mismo
mensaje: esperar `copilot_chat.mode_name=Lab Reviewer`. Despues crear una
conversacion nueva, seleccionar Lab Planner y repetir: esperar
`copilot_chat.mode_name=Lab Planner`.
En cada cambio, verificar que sigue seleccionado **`gpt-5.6-luna`**;
no asumir que cambiar de agente o de conversacion conserva el modelo.

En `received.jsonl`, comparar `gen_ai.conversation.id`: Planner y Reviewer
comparten el primero; el Planner de la nueva conversacion tiene otro.

Opcionalmente probar Lab Unmapped: debe aparecer ese valor exacto en
`copilot_chat.mode_name`. Si solo llegan spans auxiliares y falta
`invoke_agent`, detener la conclusion y revisar cliente, modalidad y version.

### Paso 9. Cerrar la demostracion

Cerrar unicamente la ventana de VS Code de pruebas. En la misma terminal:

```powershell
docker compose down
```

Despues, eliminar manualmente el directorio temporal mostrado en el paso 3
si ya no se necesita.

Conservar `demo-manual/output` como evidencia. Anotar la version de
VS Code/Copilot, el modelo solicitado y los `spanId`.
La configuracion raw es exclusivamente de laboratorio.

## Anexo A. Usage Report con GitHub App: comandos cURL

El informe y los endpoints son los mismos que con el token personal que ya
has probado. Cambia la autenticacion: no se utiliza un PAT ni el client secret.

```text
Clave privada de la App + Client ID
    -> JWT firmado localmente, de corta duracion
    -> token de la INSTALACION EN LA ENTERPRISE
    -> POST /enterprises/{enterprise}/settings/billing/reports
    -> GET de estado -> descarga de todos los CSV sin token de GitHub
```

### A.1. Preparar la App y la terminal

Antes de ejecutar cURL, tener una GitHub App con **Enterprise billing: Read**
concedido e **instalada en la enterprise**. Instalarla solo en una organizacion
no sirve para estos endpoints. El ID de la instalacion se obtiene en A.3;
no es el App ID ni el Client ID.

Para una automatizacion de lectura, es preferible una App dedicada con solo
ese permiso empresarial. **Los tokens de instalaciones enterprise no permiten
reducir permisos al generarlos:** reciben todos los permisos empresariales
concedidos a la instalacion. No anadir `permissions`, `repositories` ni
`repository_ids` al POST del token para intentar limitarlos.

Un enterprise owner debe completar la instalacion. Si la enterprise no aparece
entre los destinos, revisar los permisos empresariales y la visibilidad de la
App: las Apps con esos permisos deben ser publicas o internas para instalarse
en una enterprise. No basta con autorizar la App en la cuenta de una persona.

**Estado del producto:** la API de exportacion de informes es GA, pero la
[documentacion de instalaciones enterprise](https://docs.github.com/en/enterprise-cloud@latest/apps/using-github-apps/installing-a-github-app-on-your-enterprise)
sigue indicando **public preview** para esa modalidad de instalacion.
Son dos alcances distintos.

Se necesitan Bash, `curl`, `jq` y `openssl`, ademas del **Client ID** y del
archivo `.pem` de la clave privada de la App. La clave se queda en tu equipo:
no se envia a GitHub, no se pega en el chat y no se incluye en el paquete.
Proteger ese archivo y no compartir la pantalla mostrando su contenido.

Abrir una terminal Bash y ejecutar los bloques en orden, en la misma sesion.
Sustituir los tres valores de ejemplo. Esta secuencia es para GitHub.com:

```bash
set +x
set -euo pipefail
umask 077
mkdir demo-api-github-app
cd demo-api-github-app

APP_CLIENT_ID="CLIENT-ID-DE-TU-GITHUB-APP"
APP_PRIVATE_KEY_FILE="/ruta/segura/tu-github-app.private-key.pem"
ENTERPRISE="SLUG-DE-TU-ENTERPRISE"
API_BASE="https://api.github.com"
API="${API_BASE}/enterprises/${ENTERPRISE}/settings/billing/reports"
PAGE=1
```

No usar `curl -v`, `--trace`, `set -x` ni imprimir las variables con secretos.
Los comandos pasan la cabecera de autorizacion por la entrada estandar
(`--header @-`) para no poner su valor en los argumentos del proceso `curl`.
Si un comando falla, detenerse y resolverlo antes de seguir.

### A.2. Firmar un JWT y comprobar que identifica a la App correcta

cURL no firma JWT: esta operacion local la hace OpenSSL. Se usa `RS256`,
`iat` un minuto en el pasado y `exp` nueve minutos en el futuro, dentro del
maximo de diez minutos documentado. El Client ID es el `iss` recomendado.

```bash
base64url() {
  openssl base64 -A | tr '/+' '_-' | tr -d '='
}

NOW="$(date +%s)"
JWT_HEADER="$(printf '%s' '{"alg":"RS256","typ":"JWT"}' | base64url)"
JWT_PAYLOAD="$(jq -cn \
  --arg iss "$APP_CLIENT_ID" \
  --argjson iat "$((NOW - 60))" \
  --argjson exp "$((NOW + 540))" \
  '{iat: $iat, exp: $exp, iss: $iss}' | base64url)"
JWT_SIGNATURE="$(printf '%s' "${JWT_HEADER}.${JWT_PAYLOAD}" |
  openssl dgst -sha256 -sign "$APP_PRIVATE_KEY_FILE" | base64url)"
APP_JWT="${JWT_HEADER}.${JWT_PAYLOAD}.${JWT_SIGNATURE}"
```

Comprobar la identidad de la App sin mostrar el JWT:

```bash
curl --silent --show-error --fail-with-body \
  "$API_BASE/app" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header @- \
  --output app.json <<EOF
Authorization: Bearer ${APP_JWT}
EOF

jq '{id, slug, client_id}' app.json
```

Resultado esperado: **HTTP 200** y los datos de tu App. Para el JWT, usar
`Bearer`, no `token`. Este JWT sirve para autenticar a la App y obtener el
token de instalacion; **no es el token que se envia al endpoint de billing**.

### A.3. Encontrar la instalacion de esa App en tu enterprise

```bash
curl --silent --show-error --fail-with-body \
  "$API_BASE/app/installations?per_page=100&page=$PAGE" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header @- \
  --dump-header instalaciones.headers \
  --output instalaciones.json <<EOF
Authorization: Bearer ${APP_JWT}
EOF

jq '.[] | {
  installation_id: .id,
  target_type,
  enterprise: .account.slug,
  organization_or_user: .account.login,
  permissions,
  suspended_at
}' instalaciones.json

awk 'tolower($1) == "link:" {print}' instalaciones.headers
```

Buscar la fila cuya `enterprise` coincide con el slug elegido, no una fila
de organizacion. Si no esta en esta pagina y la cabecera `Link` contiene
`rel="next"`, incrementar `PAGE` y repetir solo el GET anterior:

```bash
PAGE=$((PAGE + 1))
```

Cuando la pagina contenga la enterprise, extraer su ID. El siguiente comando
falla si no encuentra exactamente una instalacion activa; no elige la primera
organizacion como alternativa:

```bash
INSTALLATION_ID="$(jq -er --arg enterprise "$ENTERPRISE" '
  [.[] | select(
    .account.slug == $enterprise and .suspended_at == null
  )]
  | if length == 1 then .[0].id
    else error("No hay una unica instalacion enterprise activa: revisa instalacion, slug y paginacion")
    end
' instalaciones.json)"

printf 'Installation ID: %s\n' "$INSTALLATION_ID"
```

### A.4. Obtener el installation access token

El cuerpo es `{}` deliberadamente: para una instalacion enterprise no se
recortan los permisos en esta llamada. Revisarlos en la configuracion de la
App y en la respuesta.

```bash
TOKEN_RESPONSE="$(curl --silent --show-error --fail-with-body \
  --request POST \
  "$API_BASE/app/installations/$INSTALLATION_ID/access_tokens" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header "Content-Type: application/json" \
  --header @- \
  --data '{}' <<EOF
Authorization: Bearer ${APP_JWT}
EOF
)"

INSTALLATION_TOKEN="$(printf '%s' "$TOKEN_RESPONSE" |
  jq -er '.token | select(type == "string" and length > 0)')"
printf '%s' "$TOKEN_RESPONSE" | jq '{expires_at, permissions}'
unset TOKEN_RESPONSE
```

Resultado esperado: **HTTP 201**. El token dura una hora; comprobar
`expires_at`. El ejemplo mantiene el token en memoria y solo imprime la
caducidad y los permisos, no guarda el token en un JSON.

### A.5. Solicitar el informe de septiembre con el token de la App

Este POST crea **un informe nuevo**. Ejecutarlo una vez; no repetirlo mientras
se genera. Si quieres comprobar el acceso a un informe ya creado con tu PAT,
puedes omitir este POST, asignar su ID a `REPORT_ID` y pasar a A.6.

```bash
curl --silent --show-error --fail-with-body \
  --request POST "$API" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header "Content-Type: application/json" \
  --header @- \
  --data '{
    "report_type": "ai_credit",
    "start_date": "2026-09-01",
    "end_date": "2026-09-30",
    "send_email": false
  }' \
  --output creado.json \
  --write-out 'HTTP %{http_code}\n' <<EOF
Authorization: Bearer ${INSTALLATION_TOKEN}
EOF

jq '{id, report_type, start_date, end_date, status}' creado.json
REPORT_ID="$(jq -er '.id' creado.json)"
```

Resultado esperado: **HTTP 202**. El cuerpo y el proceso asincrono son los
mismos que en el apartado 3; lo que cambia es la credencial.

### A.6. Consultar el estado y descargar el CSV

```bash
curl --silent --show-error --fail-with-body \
  "$API/$REPORT_ID" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header @- \
  --output estado.json <<EOF
Authorization: Bearer ${INSTALLATION_TOKEN}
EOF

jq '{id, status, partes: ((.download_urls // []) | length)}' estado.json
```

Si esta en `processing`, esperar y repetir este GET. Si esta en `failed`,
detenerse y revisar el error. Cuando este en `completed`, ejecutar el bloque
**3.C, "Descargar todas las partes cuando termine"**, en esta misma carpeta:
usa el `estado.json` recien obtenido y no necesita cambios.

**La descarga de los CSV no lleva JWT ni installation access token.**
No mostrar ni compartir las URLs firmadas. No asumir que siempre hay una
unica parte.

Si el token caduca durante la espera, repetir A.2 y A.4 para obtener otro
token, conservar `REPORT_ID` y continuar con el GET. No hace falta crear
otra exportacion.

### A.7. Cerrar la demostracion y diagnosticar errores

Opcionalmente, revocar el token de instalacion que acabas de utilizar:

```bash
curl --silent --show-error --fail-with-body \
  --request DELETE "$API_BASE/installation/token" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2026-03-10" \
  --header @- \
  --output /dev/null \
  --write-out 'HTTP %{http_code}\n' <<EOF
Authorization: Bearer ${INSTALLATION_TOKEN}
EOF

unset APP_JWT INSTALLATION_TOKEN JWT_HEADER JWT_PAYLOAD JWT_SIGNATURE NOW
unset -f base64url
```

Resultado esperado: **HTTP 204**. Esto revoca ese token; no desinstala la App
ni elimina el informe. La clave privada sigue siendo valida y debe permanecer
protegida en su almacen de secretos.

| Error | Que revisar |
| --- | --- |
| `401` en `/app` o al generar el token | JWT caducado, reloj incorrecto, firma RS256, Client ID o clave privada de otra App. |
| No aparece la enterprise | Instalacion realmente empresarial, permisos empresariales, visibilidad de la App, slug y paginas pendientes. |
| `401` en billing | Token de instalacion caducado o uso del JWT donde correspondia el token. |
| `403` en billing | Enterprise billing: Read efectivamente concedido, instalacion correcta y ausencia de suspension. |
| `404` | Slug, installation ID o report ID equivocados; tambien puede indicar falta de acceso. |
| `429` o limite de API | Respetar `Retry-After` y cabeceras de rate limit; no repetir POSTs de exportacion automaticamente. |

Este anexo describe el flujo documentado. No se ha creado ni instalado una
GitHub App ni ejecutado una exportacion real con ella durante la preparacion
del anexo. La firma JWT, los comandos y la seleccion de la instalacion se
comprobaron contra una API local con credenciales ficticias; el resultado esta
en `output/github-app-annex-verification.json`. No sustituye a una prueba
con una App instalada realmente en tu enterprise.

## Anexo B. YAML completos para ejecutar el Collector

Estos son los dos ficheros utilizados en el paso 1 de la demostracion manual.
Guardarlos juntos, respetando exactamente los nombres `compose.yaml` y
`collector.yaml`. Antes de arrancar el servicio, crear tambien el directorio
de salida. En PowerShell:

```powershell
New-Item -ItemType Directory -Path output -Force
docker compose up -d
```

### B.1. `compose.yaml`

```yaml
name: copilot-api-reports-otel
services:
  collector:
    image: otel/opentelemetry-collector-contrib:0.162.0@sha256:39923a8e431bd1f57be82411999d389fcfe40857492e4365456d97a4c1f74be6
    command: ["--config=/etc/otelcol/config.yaml"]
    user: "${LOCAL_UID:-501}:${LOCAL_GID:-20}"
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    cpus: 0.5
    mem_limit: 256m
    pids_limit: 64
    ports:
      - "127.0.0.1::4318"
      - "127.0.0.1::13133"
    volumes:
      - "./collector.yaml:/etc/otelcol/config.yaml:ro"
      - "./output:/data"
    logging:
      driver: json-file
      options:
        max-size: "2m"
        max-file: "1"
```

### B.2. `collector.yaml`

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
processors:
  filter/invocations:
    error_mode: propagate
    traces:
      span:
        - 'attributes["gen_ai.operation.name"] != "invoke_agent"'
  transform/privacy:
    error_mode: propagate
    trace_statements:
      - keep_keys(resource.attributes, ["service.name", "service.version"])
      - keep_keys(scope.attributes, [])
      - keep_keys(span.attributes, ["gen_ai.operation.name", "copilot_chat.mode_name", "github.copilot.agent.type", "gen_ai.request.model", "gen_ai.response.model", "gen_ai.usage.input_tokens", "gen_ai.usage.output_tokens"])
      - set(span.name, "invoke_agent")
      - set(span.status.message, "")
      - set(span.events, nil)
      - set(span.links, nil)
exporters:
  file/raw:
    path: /data/received.jsonl
    append: true
    flush_interval: 100ms
  file/sanitized:
    path: /data/sanitized.jsonl
    append: true
    flush_interval: 100ms
service:
  extensions: [health_check]
  pipelines:
    traces/raw:
      receivers: [otlp]
      exporters: [file/raw]
    logs/raw:
      receivers: [otlp]
      exporters: [file/raw]
    metrics/raw:
      receivers: [otlp]
      exporters: [file/raw]
    traces/sanitized:
      receivers: [otlp]
      processors: [filter/invocations, transform/privacy]
      exporters: [file/sanitized]
  telemetry:
    logs:
      level: warn
```

Esta configuracion conserva una salida raw con fines demostrativos. Debe
revisarse antes de cualquier piloto o uso en produccion.

## Fuentes oficiales

- [API de Usage Reports y sus cuatro tipos](https://docs.github.com/en/enterprise-cloud@latest/rest/billing/usage-reports).
- [Disponibilidad general, 4 de junio de 2026](https://github.blog/changelog/2026-06-04-api-access-to-billing-usage-reports-now-generally-available/).
- [Campos, agrupacion y rangos de los informes](https://docs.github.com/en/billing/reference/billing-reports).
- [Monitorizacion de agentes](https://code.visualstudio.com/docs/agents/guides/monitoring-agents).
- [JWT de GitHub App: RS256, Client ID y caducidad](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app).
- [Autenticacion como instalacion, caducidad y restricciones de tokens enterprise](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation).
- [Listar instalaciones y crear su access token](https://docs.github.com/en/enterprise-cloud@latest/rest/apps/apps).
- [Revocar el token de instalacion](https://docs.github.com/en/enterprise-cloud@latest/rest/apps/installations#revoke-an-installation-access-token).
- [Instalacion de una GitHub App en una enterprise](https://docs.github.com/en/enterprise-cloud@latest/apps/using-github-apps/installing-a-github-app-on-your-enterprise).
- [Permisos empresariales y visibilidad de la App](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app).
- [Acceso de GitHub Apps a billing, 26 de agosto de 2026](https://github.blog/changelog/2026-08-26-github-apps-can-now-access-enterprise-billing-data/).
