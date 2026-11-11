# Documentación con navegación lateral en GitHub Pages

## Publicar

1. Sube el contenido de esta carpeta a la raíz de la rama `main` de tu repositorio.
2. Abre **Settings → Pages**.
3. En **Build and deployment**, selecciona **Deploy from a branch**.
4. Selecciona `main` y `/(root)`, y pulsa **Save**.
5. Espera a que termine la publicación.

## Personalizar el menú lateral

Edita la sección `navigation` de `_config.yml`. Por ejemplo:

```yaml
navigation:
  - title: Inicio
    url: /
  - title: Mi sección
    children:
      - title: Página uno
        url: /pagina-uno/
      - title: Página dos
        url: /pagina-dos/
```

Cada página enlazada debe tener un `permalink` que coincida con la URL del menú. Ejemplo:

```yaml
---
layout: page
title: Página uno
permalink: /pagina-uno/
---
```

El menú se adapta a pantallas pequeñas: en móviles aparece encima del contenido.
