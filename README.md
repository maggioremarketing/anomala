# Anómala · Maqueta del sitio

Maqueta HTML del sitio de Anómala, pensada para desarrollarse después como tema de WordPress.

## Cómo verla

Abrir `index.html`: es un índice con todas las páginas de la maqueta y la plantilla de WordPress que corresponde a cada una. Dentro de cada página, los menús y enlaces también funcionan.

Para ver la maqueta con un servidor local (recomendado, por las fuentes y algunas imágenes):

```bash
npx http-server -p 5173
```

y luego abrir `http://localhost:5173`.

## Estructura

- `*.html`: una página por plantilla (portada, listados, artículos, páginas).
- `style.css`: estilos del sitio, sobre Bootstrap 5.3 (cargado desde CDN).
- `script.js`: movimiento del collage de «Quiénes somos», filtros de la cartelera (disciplina y precio), términos y montos del panel «Hazte Anómala», aviso de cookies, botones de compartir e íconos aleatorios de las entradas.
- `recortes-psd/`, `iconos-anomala/` y los `*.webp` de la raíz: imágenes del collage, íconos y papel rasgado.

La versión anterior de la maqueta (v1) está en el historial de git, en el commit «Agregar la ficha de evento de la cartelera».

Las notas son los 12 contenidos de lanzamiento (Anomala_Contenidos_Maggiore.xlsx y el acta del 19 de octubre de 2026): títulos, bajadas, autores y secciones vienen de esos documentos; el cuerpo de cada nota es lorem ipsum y las fotos son referenciales, cargadas desde Unsplash. Los autores aparecen con sus iniciales mientras no haya fotos.

Desarrollado por [Maggiore](https://maggiore.cl).
