---
name: publicar-landing-cloudflare
description: Use when the user wants to publish, deploy, update or roll back the PromoYa landing page on Cloudflare, or mentions wrangler, Workers, Pages, dominio personalizado, "subir la landing", "publicar", "desplegar".
---

# Publicar landing PromoYa en Cloudflare

## Overview

La landing es estática (`index.html` + `assets/`, sin build). Se publica como **Cloudflare Worker con Static Assets**, no como Pages: Cloudflare recomienda Workers para proyectos nuevos y Pages ya no recibe funciones nuevas. Mismo costo (gratis), CDN global, dominios personalizados y rollback de versiones.

Usa Pages solo si el usuario lo pide explícitamente o ya existe un proyecto Pages que debe conservarse.

## Flujo

1. **Preflight**
   - `git status` limpio y en la rama correcta (el repo usa `dev` / `main`). Si hay cambios sin commit, avisar antes de publicar.
   - `npx wrangler --version` (usar `npx`, no requiere instalación global).
   - `npx wrangler whoami`. Si no hay sesión, pedir al usuario que ejecute `! npx wrangler login` (es interactivo, abre el navegador). Nunca pedir ni imprimir tokens.
2. **Configuración (solo si faltan)** crear en la raíz:

   `wrangler.jsonc`
   ```jsonc
   {
     "name": "promoya-landing",
     "compatibility_date": "2026-11-01",
     "assets": {
       "directory": "./dist",
       "not_found_handling": "404-page"
     }
   }
   ```
   Se publica desde `dist/` para no subir `.git`, `.claude`, etc.
3. **Preparar `dist/`** (siempre, antes de cada deploy):
   ```bash
   rm -rf dist && mkdir dist && cp index.html dist/ && cp -R assets dist/
   ```
   Añadir `dist/` y `.wrangler/` al `.gitignore` si no están. Si aparecen más archivos públicos (favicon, robots.txt, sitemap.xml), copiarlos también.
4. **Confirmar con el usuario** el nombre del Worker y la cuenta (`whoami`) antes del primer deploy: publicar es una acción externa.
5. **Deploy**
   ```bash
   npx wrangler deploy
   ```
   Devuelve la URL `https://promoya-landing.<subdominio>.workers.dev`.
6. **Verificar**: `curl -sI <url>` debe dar `200`; comprobar también un asset (`/assets/...`). Reportar la URL real y el resultado; no declarar éxito sin esta comprobación.

## Dominio personalizado

Requiere que el dominio esté en la cuenta de Cloudflare (DNS). Añadir en `wrangler.jsonc`:
```jsonc
"routes": [{ "pattern": "promoya.com", "custom_domain": true }]
```
y volver a ejecutar `npx wrangler deploy`. Para `www`, añadir otra ruta con `custom_domain` o una regla de redirección. Preguntar el dominio; no inventarlo.

## Rollback

```bash
npx wrangler versions list
npx wrangler rollback <version-id>
```

## Referencia rápida

| Necesidad | Comando |
|---|---|
| Probar local | `npx wrangler dev` |
| Publicar | `npx wrangler deploy` |
| Ver logs | `npx wrangler tail` |
| Ver versiones | `npx wrangler versions list` |
| Cuenta activa | `npx wrangler whoami` |

## Errores comunes

- **Subir el repo entero** como `directory: "./"`: expone `.git` y archivos internos. Usar `dist/`.
- **Olvidar regenerar `dist/`**: se publica una versión vieja.
- **Código 10000 / auth error**: sesión caducada, repetir `wrangler login`.
- **Worker name ya tomado**: cambiar `name` en `wrangler.jsonc`.
- **Dominio sin activar**: la zona debe estar activa en Cloudflare antes de usar `custom_domain`.
- **Secretos en el repo**: la landing no los necesita; no commitear tokens (`CLOUDFLARE_API_TOKEN` solo como variable de entorno/CI).

## CI opcional

Para publicar al hacer push a `main`, usar GitHub Actions con `cloudflare/wrangler-action` y los secretos `CLOUDFLARE_API_TOKEN` y `CLOUDFLARE_ACCOUNT_ID`. Solo configurarlo si el usuario lo pide.
