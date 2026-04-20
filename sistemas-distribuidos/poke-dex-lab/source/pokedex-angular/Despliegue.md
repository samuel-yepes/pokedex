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
<img width="1851" height="902" alt="image" src="https://github.com/user-attachments/assets/e7da0537-1a45-4be9-a2b4-4befd06ced01" />


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

## 3. Creación del Grupo de Recursos 

Antes de crear el App Service, es fundamental definir un **Grupo de Recursos**. Este actúa como un contenedor lógico que agrupa todos los servicios relacionados con el proyecto, facilitando su administración y limpieza posterior.

### Pasos en el portal de Azure

1. En la barra de búsqueda superior, busca y selecciona **"Grupos de recursos"**.
2. Haz clic en el botón **"+ Crear"**.
   <img width="1748" height="848" alt="image" src="https://github.com/user-attachments/assets/aead1825-dbd8-4359-a474-c146f1be4b3c" />
   <img width="1381" height="888" alt="image" src="https://github.com/user-attachments/assets/1a363606-edcb-44a6-828f-0744a7db6cea" />

4. En la pestaña **Datos básicos**, completa la siguiente información:
   - **Suscripción:** `Azure for Students`
   - **Grupo de recursos:** `rg-pokedex-makia`
   - **Región:** `(US) East US`
     <img width="933" height="508" alt="image" src="https://github.com/user-attachments/assets/9c1fea29-38e9-492e-869f-13ff7117c1ea" />

5. Haz clic en **"Revisar y crear"** y luego en **"Crear"**.

> **Nota:** Se recomienda utilizar la misma región (`East US`) para todos los recursos de este proyecto para minimizar la latencia y asegurar la compatibilidad entre servicios.

---

## 4. Creación del Azure App Service

### Pasos en el portal de Azure

1. Buscar **"App Services"** en el portal
2. Hacer clic en **"+ Crear"**
   <img width="1911" height="681" alt="image" src="https://github.com/user-attachments/assets/01e7f36c-f781-44fa-a855-b2442153f3fc" />

4. Configurar:
   - **Suscripción:** `Azure for Students`
   - **Grupo de recursos:** `rg-pokedex-makia` (Seleccionar el creado en el paso anterior)
   - **Nombre:** `app-pokedex-estudiante`
   - **Región:** `East US` 
   - **Plan:** `Free F1`
     <img width="826" height="911" alt="image" src="https://github.com/user-attachments/assets/744e55d3-679c-4e2e-a3da-a6c2bb5b833f" />
     

5. Hacer clic en **"Revisar y crear"** → **"Crear"**

---

## 5. Despliegue via Kudu Zip Push Deploy

 se utilizó el panel Kudu de Azure para desplegar manualmente.

### Pasos

1. Seleccionar **todo el contenido** dentro de `dist/pokedex-angular/` con `Ctrl+A`
2. Comprimir en un archivo ZIP (ej: `pokedex-angular.zip`)
3. Navegar a Kudu:
   ir al app service:
   <img width="1131" height="538" alt="image" src="https://github.com/user-attachments/assets/4fa10cc1-db7a-4983-b22b-094c2b0de0f4" />
   herramientas avanzadas:
   <img width="1049" height="882" alt="image" src="https://github.com/user-attachments/assets/ff733644-c0fd-4dac-ad53-eca57e446465" />
   luego:
   <img width="1408" height="829" alt="image" src="https://github.com/user-attachments/assets/c04c50a4-36ee-4349-8473-ebc3f3a82cd1" />



5. Ir a **Tools → Zip Push Deploy**
   <img width="896" height="589" alt="image" src="https://github.com/user-attachments/assets/0f38b389-0954-4a42-8fd4-ee1ed385f435" />

6. Arrastrar `pokedex-angular.zip` al área de carga en la ruta: `C:\home\site\wwwroot>`
   <img width="1125" height="399" alt="image" src="https://github.com/user-attachments/assets/a034c75d-17df-4076-8c2b-02c8b35353b4" />

8. Esperar el mensaje: `Deployment successful`

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
-  Content-Security-Policy
-  X-Content-Type-Options
-  X-Frame-Options
-  Strict-Transport-Security
-  Referrer-Policy
-  Permissions-Policy

---
