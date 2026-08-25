# raylogs

Analizador de logs en **streaming**, escrito en [raylang](https://github.com/roberto-ayala/raylang): lee líneas por stdin o de un archivo (sin cargar la entrada en memoria), las parsea (JSON, CSV, regex o texto plano), filtra por campos y agrega (conteos por grupo, percentiles). Estilo angle-grinder/lnav, en pequeño.

```text
$ raylogs access.log --regex '^(\S+) (\w+) (\S+) (\d+) (\d+)$' \
    --fields ip,method,path,status,ms --count-by status
status  count
------  -----
   200      2
   500      1

$ cat app.jsonl | raylogs --json --filter 'level=error' --filter 'ms>100'
{"level":"error","status":500,"ms":140.2,"path":"/b"}

$ raylogs app.jsonl --json --stats ms
field  count  min  max    avg    p50   p90    p99
-----  -----  ---  -----  -----  ----  -----  -----
ms         5    3  140.2  52.74  12.5  140.2  140.2
```

## Uso

```text
raylogs [FILE] [opciones]          FILE omitido o '-' = stdin

formato de entrada (por defecto: plano; la línea entera es el campo 'line'):
  --json                cada línea es un objeto JSON (campos de primer nivel)
  --csv [--header]      líneas CSV; --header nombra columnas con la primera línea
  --regex PATTERN       los grupos de captura se vuelven campos g1..gN
  --fields a,b,c        nombres para los grupos de captura del regex

procesado:
  --filter EXPR         repetible (AND); key=v key!=v key~re key!~re key>n key<n key>=n key<=n
  --count               imprime solo el nº de líneas que pasan los filtros
  --count-by KEY        conteo agrupado por un campo
  --top N               con --count-by: los N grupos más frecuentes (default 20; 0 = todos)
  --stats KEY           estadística numérica: count/min/max/avg/p50/p90/p99

salida:
  --output table|json   formato de agregados y passthrough (default table)
  -f, --follow          sigue leyendo el archivo al llegar a EOF (tail -f)
```

Sin agregación, raylogs imprime las líneas que pasan los filtros (como grep, pero
entendiendo campos); con `--output json` las imprime como objeto de sus campos.
Las líneas malformadas para el formato se saltan y se reportan al final por stderr.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Lectura streaming stdin/archivo (memoria acotada por línea, no por entrada) | ✅ |
| Split de líneas a nivel de BYTE (UTF-8 partido entre chunks nunca llega roto al decoder; CRLF; última línea sin `\n`) | ✅ |
| Formatos: plano, JSON lines, CSV (con/sin cabecera), regex con grupos | ✅ |
| Filtros compuestos: `= != ~ !~ > < >= <=` (numéricos y regex) | ✅ |
| `--count`, `--count-by` (+`--top`), `--stats` (percentiles nearest-rank) | ✅ |
| Salida tabla alineada (números a la derecha) o JSON | ✅ |
| `--follow` (tail -f por EVENTOS de kernel — fs.watch, raylang M115.4) | ✅ |
| Binario nativo (`ray build --native`) | ✅ |
| Tests (parser, filtros, agregación, reader con archivos reales) | ✅ 27 |
| Ventanas temporales (`--window`), agregación en vivo con `--follow` | 📋 v2 |
| Campos multilínea CSV (comillas que cruzan líneas) | ❌ fuera de v1 (parseo por línea) |

## Rendimiento (señal para PERFORMANCE.md)

200k líneas, Apple Silicon, misma máquina, mejores de 3:

| Workload | VM (`ray run`) | Nativo | awk/grep de referencia |
|---|---|---|---|
| JSON parse + count-by | 16.5 s | **0.62 s** | awk (extracción cruda por split): 0.48 s |
| regex 5 grupos + count-by | 8.75 s | **0.21 s** | grep -c (sin captura): 0.008 s |
| JSON parse + stats (sort 200k floats) | — | **0.62 s** | — |

El nativo queda en el orden de awk haciendo bastante más trabajo (parse JSON
completo + Map por grupo). La brecha VM/nativo en este workload es 27–40×.

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

1. **`sort([float])` rompe el build nativo**: compila y corre en la VM, pero el
   transpilador emite `__ray_sort<T: Ord>` y `f64` no es `Ord` en Rust
   (`error[E0277]`). [RESUELTO en raylang, PR #140: `sort([float])` compila
   nativo]; el mergesort propio de `src/agg.ray` ya no es necesario.
2. **[RESUELTO — raylang M115.4]** No hay `tail -f`: `fs.watch` existe y
   `--follow` aparca en eventos de kernel (con degradación a sleep-poll si el
   watch no se puede armar).
3. **`std/regex` no tiene grupos con nombre** (`(?P<name>...)`): los nombres se
   inyectan por CLI con `--fields`. Para un analizador de logs es la ergonomía
   que se espera.
4. **`std/csv` parsea documentos, no streams**: `parse_csv` por línea funciona,
   pero un campo entrecomillado con `\n` dentro es invisible para un lector por
   líneas. Un parser incremental (push de chunks) sería la pieza streaming.
5. Lo que SÍ estuvo a la altura: `io.read`/`fs.read_bytes` streaming con
   `sub_bytes` por octeto hacen el line-splitting limpio; `captures_str` +
   `Matcher` compilado; `parse_float` tolera espacios; la tabla de la stdlib
   (`text.pad_*`) alcanza para el render.

## Desarrollo

```sh
ray test                                  # 27 tests
ray run src/main.ray sample.jsonl --json --count-by level
ray build --native src/main.ray -o raylogs --release
```

Estructura: `src/main.ray` (CLI) · `reader.ray` (líneas en streaming) ·
`parse.ray` (formatos → Event) · `filter.ray` (expresiones) · `agg.ray`
(count-by/stats) · `render.ray` (tabla/JSON).
