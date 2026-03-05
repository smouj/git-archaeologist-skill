name: git-archaeologist
description: Analiza el historial de git para extraer la justificación del código y generar documentación histórica
version: 1.0.0
author: OpenClaw Team
tags:
  - git
  - arqueología
  - documentación
  - historial
  - análisis-de-código
maintainer: SMOUJBOT
category: development
type: research
dependencies:
  - git>=2.30
  - python3
  - gitpython>=3.1
  - jinja2
  - pyyaml
  - graphviz (opcional)
```

# Git Archaeologist

## Propósito

Casos de uso reales:
- Inducción de ingenieros a bases de código heredadas extrayendo contexto de toma de decisiones de mensajes de commit y discusiones de pull request
- Auditorías de cumplimiento: trazar cambios regulatorios a través de la evolución del código e identificar quién aprobó cambios de lógica específicos
- Triaje de bugs: determinar cuándo y por qué se introdujo una función problemática analizando blame + contexto histórico
- Revisión de arquitectura: mapear la evolución de características a lo largo del tiempo para identificar patrones de acumulación de deuda técnica
- Análisis de brechas de documentación: identificar secciones de código con registros históricos escasos que requieren documentación manual

## Alcance

Comandos concretos que esta habilidad proporciona:
- `git-archaeology dig <commit_range> --output=markdown`
- `git-archaeology blame --enhanced <file_path> --since="2024-01-01"`
- `git-archaeology timeline --feature="auth" --include-merges`
- `git-archaeology authorship <directory> --heatmap`
- `git-archaeology rationale <commit_hash> --extract-prs`
- `git-archaeology report --format=html --template=compliance`

Los parámetros son concretos: `--since`, `--until`, `--author`, `--regex`, `--max-depth`, `--prune-tests`

## Proceso de Trabajo

1. Preparación del repositorio:
   - Asegurar directorio de trabajo limpio: `git status --porcelain | grep -q . && exit 1`
   - Obtener todos los remotos: `git fetch --all --prune`
   - Identificar alcance del análisis: archivo individual, directorio o rango de commits

2. Extraer historial crudo:
   - Usar `git log --all --pretty=format:'%H|%an|%ae|%ad|%s' --date=iso --no-merges` para commits
   - Aumentar con `git show --stat --name-only <commit>` para cambios de archivos por commit
   - Obtener metadatos de PR si GitHub CLI disponible: `gh pr list --json number,title,body,author`

3. Enriquecimiento de contexto:
   - Correlacionar commits con PRs usando patrones de commit convencionales (feat:, fix:, chore:)
   - Ejecutar `git blame -C -C <file>` para detectar movimiento de código entre archivos
   - Extraer comentarios de code review vía API de GitHub cuando se encuentran PRs

4. Detección de patrones:
   - Identificar "código huérfano" (sin commits en >6 meses)
   - Detectar adiciones grandes de un solo commit (>500 líneas) para revisión
   - Marcar commits con mensajes vagos ("fix bug", "update") para seguimiento

5. Generación de documentación:
   - Renderizar plantillas Jinja2 con metadatos descubiertos
   - Incluir snippets de código de estados antes/después
   - Agregar visualizaciones de línea de tiempo si graphviz presente

6. Validación:
   - Verificar que todos los commits referenciados aún existen en el repositorio
   - Verificar que las rutas de archivos no han sido eliminadas
   - Cruzar referencia de consistencia de email de autor

## Reglas de Oro

1. Nunca reescribir el historial. Todas las lecturas son de solo lectura; sin `git rebase` o `filter-branch`.
2. Preservar mensajes de commit originales textualmente; no resumir ni parafrasear.
3. Incluir siempre hashes de commit en las salidas para trazabilidad.
4. Manejar repos grandes con elegancia: paginar resultados, no cargar log completo en memoria.
5. Cuando el enlace de PR falle, registrar advertencia pero continuar el análisis.
6. Respetar .gitignore y excluir archivos binarios del análisis detallado.
7. Para repos privados, nunca cachear tokens de auth; usar auth de GitHub CLI basada en sesión.
8. Reportar incertidumbres: "No se puede determinar la justificación, el mensaje del commit carece de contexto" es una salida válida.
9. Limitar llamadas API automatizadas a GitHub a límites de tasa (máx 5000/hora).
10. Siempre verificar raíz del repositorio antes de ejecutar; prevenir escape a directorios padres.

## Ejemplos

Ejemplo 1: Arqueología básica en una característica
```
Entrada: git-archaeology dig --since="2024-01-01" --output=markdown
Salida: archaeology-report-2024-01-01-to-2024-12-31.md conteniendo:
- 234 commits analizados
- 47 PRs enlazados
- Principales colaboradores: alice@example.com (32%), bob@example.com (28%)
- Archivos huérfanos: src/legacy/utils.py (última modificación 2023-05-12)
- Cambio más grande: commit abc123 agregó 847 líneas en src/auth/
```

Ejemplo 2: Blame mejorado con seguimiento de movimiento
```
Entrada: git-archaeology blame --enhanced src/auth/login.py --since="2024-06-01"
Salida:
Archivo: src/auth/login.py (85% original, 15% movido/copiado)
Línea 45-67: movido desde src/auth/old_session.py en commit def456 (2024-07-21) por bob@example.com
Justificación: PR #342 "Refactor session handling" - "Moving to stateless JWT tokens"
```

Ejemplo 3: Generación de línea de tiempo para cumplimiento
```
Entrada: git-archaeology timeline --feature="payment" --include-merges --prune-tests
Salida: payment-evolution.svg timeline mostrando:
- Hito: commit 789xyz (2024-03-15) agregó integración Stripe
- Regresión: commit 234abc (2024-05-02) revirtió a PayPal debido a cumplimiento PCI
- Feature flag: commit 567def (2024-08-10) introdujo soporte multi-vendedor
```

Ejemplo 4: Mapa de calor de autoría
```
Entrada: git-archaeology authorship src/components/ --heatmap
Salida: authorship-heatmap.csv
Componente,autores,último_toque,propietarios_complejidad
Button.tsx,alice (90%),2024-12-01,solo
Modal.tsx,alice (60%),bob (40%),2024-11-15,compartido
```

Ejemplo 5: Extracción de justificación para commit específico
```
Entrada: git-archaeology rationale abc123 --extract-prs
Salida:
Commit: abc123 "Fix login race condition"
PR: #456 titulado "Fix concurrent login edge case"
Autor: alice@example.com
Revisores: bob@example.com, charlie@example.com
Excerpto de discusión: "This lock-free approach uses atomic CAS operation; benchmark shows 15% throughput increase under load."
Diff de código: (+12 -8)
```

## Comandos de Rollback

Dado que esta habilidad es de solo lectura, el rollback es simplemente eliminación de artefactos generados:

- `rm -f archaeology-report-*.md archaeology-report-*.html *.csv *.svg`
- `git restore --staged .` (si algún archivo temporal fue accidentalmente staged)
- Matar cualquier proceso persistente: `pkill -f git-archaeology` (si está daemonizado)

El estado del repositorio git nunca es modificado; el rollback es solo limpieza.

## Variables de Entorno

- `GIT_ARCHAEOLOGY_GITHUB_TOKEN`: opcional, para metadatos de PR (cae en gh cli)
- `GIT_ARCHAEOLOGY_MAX_COMMITS`: default 10000, profundidad máxima de historial
- `GIT_ARCHAEOLOGY_TEMPLATE_DIR`: ruta a plantillas Jinja2 personalizadas
- `GIT_ARCHAEOLOGY_VERBOSE`: establecer a 1 para logging de depuración

## Dependencias y Requisitos

Sistema:
- Python 3.8+
- Git 2.30+ (para `git log --date=iso` y `--name-only`)

Paquetes Python (instalar via `pip install git-archaeology`):
- GitPython>=3.1.30
- Jinja2>=3.0
- PyYAML>=5.4
- click (wrapper CLI)
- python-dateutil (parsing de fechas)
- tabulate (salida tabular)

Opcional:
- github3.py o gh (GitHub CLI) para integración PR
- graphviz para SVGs de línea de tiempo

## Pasos de Verificación

Después de instalar la habilidad:
1. Ejecutar `git-archaeology --version` → debería imprimir 1.0.0
2. En un repo de prueba: `git init && touch foo && git add . && git commit -m "initial"` luego `git-archaeology dig` → debería producir reporte vacío con 1 commit
3. Simular historial multi-commit: agregar 3 commits, ejecutar `git-archaeology blame --enhanced foo` → debería detectar sin movimiento y reportar 100% original
4. Probar enlace PR (si token GitHub establecido): crear PR con commit convencional, ejecutar `git-archaeology dig --since="1 day ago"` → debería incluir número de PR en la salida

## Solución de Problemas

Problema: "fatal: not a git repository"
- Solución: Asegurar que el directorio actual está dentro de un repo git. Usar `git rev-parse --show-toplevel` para verificar.

Problema: "GitPython import failed"
- Solución: Instalar con `pip install GitPython`. Verificar versión de Python: `python3 --version`.

Problema: "rate limit exceeded" de API de GitHub
- Solución: Establecer `GIT_ARCHAEOLOGY_GITHUB_TOKEN` con un token de acceso personal (scope repo:read). O deshabilitar enlace PR con `--no-prs`.

Problema: "MemoryError en repo grande"
- Solución: Establecer `GIT_ARCHAEOLOGY_MAX_COMMITS=5000` para limitar profundidad. Usar `--since` para narrow rango.

Problema: Generación de SVG de línea de tiempo falla
- Solución: Instalar graphviz: `apt-get install graphviz` o `brew install graphviz`. Asegurar ejecutable `dot` en PATH.

Problema: Blame no detecta movimiento
- Solución: Verificar que `git blame -C -C` funciona manualmente. Asegurar que el código realmente se movió; la detección de copia requiere chunks idénticos.

Problema: Fechas en reporte son UTC, se necesita timezone local
- Solución: Establecer `TZ=America/Los_Angeles` (o tz deseado) antes de ejecutar. Las fechas de git log se convierten usando timezone del sistema.

Problema: Errores de codificación UTF-8 en mensajes de commit
- Solución: Asegurar locale es UTF-8: `export LC_ALL=en_US.UTF-8`. Git almacena bytes crudos; Python usa locale encoding por defecto.
```