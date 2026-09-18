# Panel de noticias — Nuevo León

Dashboard que muestra las noticias más relevantes de Nuevo León, organizadas por
Seguridad, Política y gobierno, Economía, Eventos y cultura, y Sociedad y
deportes. Se actualiza solo tres veces al día: un GitHub Action recolecta
titulares, se los manda a Claude para que elija los más relevantes y los resuma,
y regenera `index.html`.

**Enfoque editorial:** la prioridad es política/gobierno y seguridad en el
estado. El clima se descarta siempre. El futbol (Tigres, Rayados) está limitado
a propósito porque de otro modo se come la categoría de deportes.

Ojo: los medios de Monterrey cubren mucha más nota nacional e internacional que
los de Baja California. Este panel depende del filtro de curación más de lo que
dependía el de BC, y varias fuentes se apuntan a secciones en vez de portadas
justamente por eso.

## Puesta en marcha

### 1. Crear el repo y subir estos archivos

Sube todo respetando la estructura de carpetas:

```
scripts/fetch_news.py
scripts/curate_and_render.py
templates/index_template.html
.github/workflows/update.yml
requirements.txt
README.md
```

### 2. Activar GitHub Pages

Settings → Pages → "Build and deployment" → Deploy from a branch → rama `main`,
carpeta `/` (root). En un par de minutos el panel queda en
`https://TU_USUARIO.github.io/NOMBRE-DEL-REPO/`

### 3. Configurar la API key de Claude

Settings → Secrets and variables → Actions → New repository secret.
Nombre: `ANTHROPIC_API_KEY`. Valor: tu API key de console.anthropic.com.

Puedes usar la misma key que el panel de Baja California. Ojo con eso: **dos
paneles compartiendo una key consumen el doble de crédito.** Con Haiku sigue
siendo de centavos al mes, pero revisa el saldo la primera semana.

### 4. Dar permiso de escritura al Action

Settings → Actions → General → Workflow permissions → "Read and write
permissions". Sin esto el Action no puede hacer commit del `index.html`.

### 5. Primera corrida

Actions → "Actualizar panel de noticias — Nuevo León" → Run workflow.

**La primera corrida es la lenta**: el autodescubrimiento prueba varias rutas de
RSS en cada fuente que no trae feed declarado. Lo que encuentra queda guardado en
`data/feeds.json` y de ahí en adelante ya no vuelve a probar.

En el log busca:

- `[descubierto]` — la fuente sí tenía RSS, va a ser estable
- `[sin RSS]` — cayó a scraping del HTML, más frágil
- `[aviso] ... 403` — el sitio bloquea a GitHub Actions; no se arregla con código
- La lista de ✓ / ✗ al final, y el conteo por categoría del segundo paso

Si borras `data/feeds.json`, el autodescubrimiento vuelve a correr desde cero.
Útil cuando cambias el User-Agent o crees que una fuente ya se destrabó.

## Cómo está armado

- `scripts/fetch_news.py` — junta titulares. Lee RSS cuando existe (declarado o
  autodescubierto) y scrapea el HTML cuando no. Filtra a solo notas de hoy con
  tres criterios: fecha del RSS, fecha en la URL, e historial de URLs vistas.
  Guarda todo en `raw_items.json`.
- `scripts/curate_and_render.py` — manda hasta 180 titulares a Claude (Haiku),
  pide hasta 8 notas nuevas por categoría, acumula el día en `data/today.json`
  con tope de 15 por categoría, y regenera `index.html` desde la plantilla.
- `.github/workflows/update.yml` — corre ambos scripts y publica los cambios.

## Fuentes

16 fuentes:

**Con foco político:** Código Magenta, Reporte Índigo.
**Diarios y portales:** El Horizonte, Posta, ABC Noticias, INFO7, Hora Cero,
EIT Media, El Porvenir (secciones Local y Justicia por separado).
**Televisión:** Canal 28 (SRTVNL, del gobierno del estado).
**Secciones de nacionales:** Milenio Monterrey, Telediario Monterrey, TV Azteca,
El Sol de México (etiqueta Monterrey).
**En prueba:** El Norte.

### Sobre El Norte

Es el diario de referencia del estado, de Grupo Reforma, con muro de pago duro.
Está incluido porque para el panel basta el titular y la liga, y los titulares
de portada se ven sin suscripción. **Al abrir una nota desde el panel vas a
topar con el muro.** Si en el log sale 403 o con cero notas, significa que
también bloquea el scraping: se comenta la fuente y se lee a mano.

### Lo que NO está y por qué

- **Luminaria Media** — eran dos periodistas de investigación con beca del
  Border Hub, pero su sitio hoy es de consultoría en comunicación y PR, no de
  periodismo. Ya no funciona como fuente.
- **Facebook e Instagram** — requieren login, no son accesibles por script. Si
  un medio solo existe ahí, queda fuera.
- Cualquier fuente que en el primer log salga con 403 o con cero notas sin
  error. El 403 casi siempre es bloqueo por IP de datacenter y no tiene arreglo
  desde GitHub Actions; cero notas sin error suele ser un sitio hecho en
  JavaScript, que tampoco se puede scrapear así.

## Limitaciones

- El scraping es frágil: si Milenio, Telediario o TV Azteca rediseñan su sitio,
  esa fuente deja de traer notas hasta que se ajuste el código. Revisa los logs
  de vez en cuando.
- El acumulado se borra a medianoche hora de Monterrey, así que la corrida de
  9am arranca con el panel casi vacío. Si molesta, hay que cambiar el filtro de
  fecha para aceptar 48 horas en vez de solo hoy.
- Buena parte de las notas pasan el filtro "sin fecha" y dependen del historial
  de URLs para no repetirse. Es el criterio más débil de los tres.
- El resumen lo genera un modelo de lenguaje: puede cometer errores o perder
  matices. Trátalo como un primer vistazo, no como fuente única.
- Las fuentes se eligieron por investigación, no por verificación una por una.
  El primer log dirá cuáles funcionan de verdad.
