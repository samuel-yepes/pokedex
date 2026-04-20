#  PokeDex — Despliegue en Azure App Service


**Estudiante:** Samuel Yepes
**Curso:** Sistemas Distribuidos
**Fecha:** 14/02/2026
**URL pública:** https://app-pokedex-estudiante-cqbvd9g8cmg0e5gy.eastus-01.azurewebsites.net/

---

##  Cuenta en la Nube — Azure for Students

### Paso 1: Activar Azure for Students

1. Ingresar a [https://azure.microsoft.com/es-es/free/students/](https://azure.microsoft.com/es-es/free/students/)
2. Hacer clic en **"Activar ahora"**
3. Iniciar sesión con el **correo institucional** (formato `@tecnocomfenalco.edu.co`)
4. Completar la verificación de identidad estudiantil
5. Aceptar los términos y condiciones
6. La cuenta se activa con **$100 USD de crédito gratuito** sin necesidad de tarjeta de crédito

### Paso 2: Acceder al Portal de Azure

1. Ir a [https://portal.azure.com](https://portal.azure.com)
2. Iniciar sesión con las credenciales institucionales
3. Verificar que la suscripción activa sea **"Azure for Students"**

### Paso 3: Crear un Grupo de Recursos

1. En el portal, buscar **"Grupos de recursos"**
2. Hacer clic en **"+ Crear"**
3. Configurar:
   - **Suscripción:** Azure for Students
   - **Nombre:** `rg-pokedex-estudiante`
   - **Región:** East US *(nota: East US 2 puede estar bloqueada por política institucional)*
4. Hacer clic en **"Revisar y crear"** → **"Crear"**

---

##  Reflexión Técnica

### 1. ¿Qué vulnerabilidades previenen los encabezados implementados?

| Encabezado | Vulnerabilidad que previene |
|---|---|
| `Content-Security-Policy` | Ataques XSS (Cross-Site Scripting), inyección de scripts maliciosos |
| `Strict-Transport-Security` | Ataques de degradación a HTTP, interceptación de tráfico (MITM) obligando al navegador a usar siempre HTTPS|
| `X-Content-Type-Options: ` | MIME sniffing , evitando que archivos de texto o imágenes sean interpretados como scripts ejecutables. |
| `X-Frame-Options: ` | Clickjacking, carga de la app en iframes de terceros |
| `Referrer-Policy: n` | Fuga de información sensible en headers de referencia |
| `Permissions-Policy` | Acceso no autorizado a hardware del dispositivo (cámara, micrófono, geolocalización) |

### 2. ¿Qué aprendiste sobre la relación entre despliegue y seguridad web?

El despliegue no termina cuando la aplicación es accesible públicamente. La seguridad debe ser parte integral del proceso desde el inicio. Configurar correctamente los headers HTTP es una capa de protección fundamental que no requiere cambios en el código de la aplicación, sino en la configuración del servidor. Un despliegue sin estas cabeceras expone innecesariamente a los usuarios a vectores de ataque comunes y bien documentados.

### 3. ¿Qué desafíos encontraste en el proceso?

- **Restricción de regiones:** La suscripción institucional bloqueaba `eastus2`. Fue necesario identificar las regiones permitidas y recrear el recurso.
- **Content Security Policy vs Azure:** Azure App Service inyectaba su propio header CSP más restrictivo que sobreescribía el configurado en `web.config`. La solución fue agregar `<remove>` antes de cada `<add>` en los headers.
- **Base-href incorrecto:** La aplicación Angular estaba compilada con `--base-href=/pokedex-angular/`, lo que causaba que las imágenes de la lista de Pokémon no cargaran. Se requirió identificar la variable `imagesPath` en `environment.prod.ts` y recompilar con `--configuration production --base-href=/`.
- **Estructura del ZIP:** Al hacer deploy manual via Kudu Zip Push Deploy, comprimir la carpeta en lugar de su contenido resultaba en que los archivos quedaban en un subdirectorio dentro de `wwwroot`.
