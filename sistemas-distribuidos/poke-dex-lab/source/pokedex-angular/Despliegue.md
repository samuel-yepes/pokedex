#  Despliegue.md — PokeDex en Azure App Service

**Proyecto:** PokeDex Angular
**Plataforma:** Microsoft Azure — App Service 
**Repositorio:** https://github.com/samuel-yepes/pokedex
**URL desplegada:** https://app-pokedex-estudiante-cqbvd9g8cmg0e5gy.eastus-01.azurewebsites.net/

---

## 1. Preparación del Entorno Local

### Requisitos previos
- Node.js >= 14
- Angular CLI (`npm install -g @angular/cli`)
- Cuenta en Azure for Students activa

### Clonar el repositorio

```bash
git clone https://github.com/samuel-yepes/pokedex.git
cd pokedex/sistemas-distribuidos/poke-dex-lab/source/pokedex-angular
npm install
```

---

## 2. Compilación para Producción

Se utilizó el siguiente comando para generar el bundle de producción con la ruta base correcta para Azure:

```bash
npm run build -- --configuration production --base-href=/
```

>  **Importante:** No omitir `--configuration production`. Sin esta flag, Angular usa `environment.ts` en lugar de `environment.prod.ts`, lo que causa rutas de assets incorrectas.

**Salida esperada en `dist/pokedex-angular/`:**

```
assets/
index.html
main.xxxxxxxx.js
polyfills.xxxxxxxx.js
styles.xxxxxxxx.css
runtime.xxxxxxxx.js
web.config
...
```

---

## 3. Configuración de Seguridad — web.config

Se creó/modificó el archivo `web.config` dentro de `dist/pokedex-angular/` con los headers HTTP de seguridad y la regla de rewrite para que Angular maneje el enrutamiento SPA:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <httpProtocol>
      <customHeaders>
        <remove name="X-Powered-By" />
        <remove name="Content-Security-Policy" />
        <add name="Content-Security-Policy" value="default-src 'self'; connect-src 'self' https://beta.pokeapi.co https://pokeapi.co wss://beta.pokeapi.co; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com data:; img-src 'self' data: https://*.githubusercontent.com https://*.pikaserve.net https://*.pokemon.com;" />
        <remove name="X-Content-Type-Options" />
        <add name="X-Content-Type-Options" value="nosniff" />
        <remove name="X-Frame-Options" />
        <add name="X-Frame-Options" value="DENY" />
        <remove name="Strict-Transport-Security" />
        <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains" />
        <remove name="Referrer-Policy" />
        <add name="Referrer-Policy" value="no-referrer" />
        <remove name="Permissions-Policy" />
        <add name="Permissions-Policy" value="geolocation=(), microphone=(), camera=()" />
      </customHeaders>
    </httpProtocol>
    <rewrite>
      <rules>
        <rule name="Angular" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="index.html" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

>  **Nota crítica:** Cada header requiere un `<remove>` antes del `<add>`. Azure App Service puede inyectar sus propios headers, y sin el `<remove>`, los headers se duplican aplicándose el más restrictivo.

---

## 4. Creación del Azure App Service

### Pasos en el portal de Azure

1. Buscar **"App Services"** en el portal
2. Hacer clic en **"+ Crear"**
3. Configurar:
   - **Suscripción:** Azure for Students
   - **Grupo de recursos:** `rg-pokedex-estudiante`
   - **Nombre:** `app-pokedex-estudiante`
   - **Publicar:** Código
   - **Pila de runtime:** Node 18 LTS
   - **Región:** East US *(ver error #1 abajo)*
   - **Plan:** Free F1
4. Hacer clic en **"Revisar y crear"** → **"Crear"**

---

## 5. Despliegue via Kudu Zip Push Deploy

Al no tener GitHub Actions configurado, se utilizó el panel Kudu de Azure para desplegar manualmente.

### Pasos

1. Seleccionar **todo el contenido** dentro de `dist/pokedex-angular/` con `Ctrl+A`
2. Comprimir en un archivo ZIP (ej: `deploy.zip`)
3. Navegar a Kudu:
   ```
   https://app-pokedex-estudiante-cqbvd9g8cmg0e5gy.scm.azurewebsites.net
   ```
4. Ir a **Tools → Zip Push Deploy**
5. Arrastrar `deploy.zip` al área de carga
6. Esperar el mensaje: `Deployment successful`

> **Importante:** Comprimir el **contenido** de la carpeta, no la carpeta en sí. Al abrir el ZIP, `index.html` debe verse directamente, no dentro de una subcarpeta.

---

## 6. Errores Encontrados y Soluciones

### Error 1 — Región bloqueada por política institucional

**Mensaje:**
```json
{
  "code": "RequestDisallowedByAzure",
  "message": "Resource was disallowed by Azure: This policy maintains a set of best available regions..."
}
```

**Causa:** La suscripción institucional tiene restricciones de región. `eastus2` no estaba permitida.

**Solución:** Eliminar el recurso y recrearlo en la región `East US` (sin el "2").

---

### Error 2 — CSP bloqueaba Google Fonts y PokeAPI

**Mensaje en consola:**
```
Loading the stylesheet 'https://fonts.googleapis.com/...' violates Content Security Policy directive: "style-src 'self' 'unsafe-inline'"
Connecting to 'https://beta.pokeapi.co/graphql/v1beta' violates Content Security Policy directive: "default-src 'self'"
```

**Causa:** Azure App Service inyectaba su propio header CSP más restrictivo antes del definido en `web.config`.

**Solución:** Agregar `<remove name="Content-Security-Policy" />` antes del `<add>` en `web.config` para eliminar el header previo.

---

### Error 3 — Imágenes de Pokémon no aparecían en la lista

**Síntoma:** Las cartas de la lista mostraban nombre y número pero no la imagen del sprite. En la página de detalle sí aparecía.

**Causa 1:** El build usaba `--base-href=/pokedex-angular/`, lo que generaba rutas como `/pokedex-angular/assets/images/pokemon-green.png` que no existían en Azure.

**Causa 2:** La variable `imagesPath` en `environment.prod.ts` tenía hardcodeado `/pokedex-angular/assets/images`.

**Solución:**
1. Cambiar en `src/environments/environment.prod.ts`:
   ```typescript
   imagesPath: '/assets/images',  // era '/pokedex-angular/assets/images'
   ```
2. Recompilar con:
   ```bash
   npm run build -- --configuration production --base-href=/
   ```

---

### Error 4 — Archivos desplegados en subcarpeta de wwwroot

**Síntoma:** Kudu mostraba una carpeta `pokedex-angular/` dentro de `wwwroot` en lugar de los archivos directamente.

**Causa:** Se comprimió la carpeta `pokedex-angular/` completa en lugar de su contenido.

**Solución:** Entrar dentro de la carpeta `dist/pokedex-angular/`, seleccionar todo con `Ctrl+A` y comprimir desde adentro.

---
**Headers presentes:**
- ✅ Content-Security-Policy
- ✅ X-Content-Type-Options
- ✅ X-Frame-Options
- ✅ Strict-Transport-Security
- ✅ Referrer-Policy
- ✅ Permissions-Policy

---
