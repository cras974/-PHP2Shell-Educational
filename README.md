# 🔐 PHP2Shell

## Dos vulnerabilidades críticas de ejecución remota en PHP

> **Fecha de referencia:** 22 de octubre de 2026  
> **Vulnerabilidades:** CVE-2026-109618 y CVE-2026-109619  
> **Gravedad:** 🔴 Crítica — CVSS 4.0: 9,2/10

---

> [!CAUTION]
> Estas vulnerabilidades pueden permitir la ejecución remota de código **sin autenticación**. Si administras un servicio PHP expuesto a Internet, actualizar debe considerarse una prioridad inmediata.

## 📌 En pocas palabras

| Pregunta | Respuesta |
|---|---|
| ¿Qué ocurre? | PHP puede tratar memoria o estado interno inválido como legítimo. |
| ¿Cuál es el impacto? | Posible ejecución remota de código con los permisos del servidor web. |
| ¿Hace falta iniciar sesión? | No. |
| ¿Afecta solo a una aplicación? | No; el fallo está en el intérprete de PHP. |
| ¿Existe un exploit? | Sí, se ha demostrado la explotación y existen pruebas de concepto en el ámbito de la investigación de seguridad. |
| ¿Cuál es la solución? | Actualizar PHP a una versión corregida y reiniciar todos sus procesos. |

## 🧭 Contenido

- [Resumen](#-resumen)
- [Cómo funciona](#-cómo-funciona)
- [Sobre el exploit](#-sobre-el-exploit)
- [Versiones corregidas](#-versiones-corregidas)
- [Comprobaciones defensivas](#-comprobaciones-defensivas)
- [Mitigaciones temporales](#-mitigaciones-temporales)
- [Plan de respuesta](#-plan-de-respuesta-recomendado)
- [Descargo de responsabilidad](#-divulgación-responsable-y-descargo-de-responsabilidad)

## 🧩 Resumen

El 21 de octubre de 2026, el equipo de PHP publicó actualizaciones de emergencia para corregir dos vulnerabilidades críticas conocidas informalmente como **PHP2Shell**. Los fallos pueden permitir que un atacante remoto ejecute código en un servidor vulnerable sin disponer de credenciales.

El problema resulta especialmente preocupante porque se encuentra en el propio intérprete de PHP. Una aplicación puede estar correctamente programada y, aun así, quedar expuesta por la versión o configuración del entorno que la ejecuta.

Los entornos potencialmente afectados incluyen:

- PHP-FPM.
- Apache con `mod_php`.
- `php-cgi`.
- FrankenPHP.
- LiteSpeed.

Según el aviso oficial, el riesgo demostrado es mayor en Apache con `mod_php`, donde determinadas rutas pueden activarse antes de que comience a ejecutarse el código de la aplicación.

## 🧠 Cómo funciona

### CVE-2026-109618: corrupción de memoria por OOM

PHP puede encontrarse con un error de agotamiento de memoria —*Out of Memory* u OOM— mientras todavía está construyendo estructuras internas asociadas a una petición HTTP. En ciertas circunstancias, la fase de limpieza puede interpretar datos incompletos o inválidos como si fueran legítimos.

El resultado potencial abarca corrupción de memoria, exposición de información y ejecución remota de código con los permisos del proceso web.

### CVE-2026-109619: estado global obsoleto

Determinados datos internos pertenecientes a una petición pueden permanecer referenciados después de que PHP haya liberado su memoria. Si el mismo proceso atiende una petición posterior, podría reutilizar ese estado obsoleto antes de reinicializarlo correctamente.

Esta explotación requiere encadenar una operación que deje el estado inconsistente con otra que lo consuma. Se han confirmado rutas relacionadas con errores fatales, trazas y el progreso de subida de sesiones.

### Flujo conceptual

```text
Petición remota
      │
      ▼
Procesamiento interno de PHP
      │
      ├── Agotamiento o corrupción de memoria
      └── Estado interno que sobrevive indebidamente
      │
      ▼
Uso posterior de datos inválidos
      │
      ▼
Posible ejecución de código con permisos del proceso web
```

## 🧪 Sobre el exploit

La explotación ha sido demostrada por investigadores contra configuraciones como Apache con `mod_php` y PHP-FPM. En términos generales, las cadenas conocidas manipulan el procesamiento temprano de peticiones y el ciclo de vida de los workers para provocar que PHP utilice memoria o estado interno inválido.

Existen pruebas de concepto y se ha informado de actividad temprana relacionada con estas vulnerabilidades. No se reproduce aquí código, cargas útiles ni una guía operativa de explotación, ya que podrían reutilizarse directamente contra servidores expuestos. Para investigación autorizada, cualquier validación debe realizarse en un laboratorio aislado, con datos ficticios, instantáneas recuperables y un alcance documentado.

> [!IMPORTANT]
> Un fallo de autenticación en la aplicación no es necesario: determinadas rutas se activan antes de que el código de WordPress, Laravel u otra aplicación pueda intervenir.

## ✅ Versiones corregidas

| Rama de PHP | Versión corregida mínima |
|---|---:|
| 8.2 | 8.2.35 |
| 8.3 | 8.3.36 |
| 8.4 | 8.4.27 |
| 8.5 | 8.5.12 |

PHP 8.1 y las ramas anteriores ya no reciben versiones binarias ordinarias del proyecto. Algunos distribuidores y proveedores LTS ofrecen parches retroportados, por lo que el número de versión por sí solo puede no ser suficiente: es necesario consultar el aviso de seguridad del proveedor.

## 🛡️ Comprobaciones defensivas

Los siguientes ejemplos sirven para comprobar y proteger sistemas propios o expresamente autorizados. No incluyen una carga útil de explotación.

### 1. Comprobar la versión instalada

```bash
php --version
```

En un contenedor:

```bash
docker exec mi-contenedor-php php --version
```

Si la versión está por debajo de la corregida para su rama, debe actualizarse. En distribuciones que incorporen parches retroportados también hay que revisar el boletín del paquete instalado.

### 2. Identificar cómo se está ejecutando PHP

```bash
php -i | grep 'Server API'
```

También puede revisarse si existen procesos PHP-FPM o Apache:

```bash
ps aux | grep -E 'php-fpm|apache2|httpd' | grep -v grep
```

Esta información ayuda a priorizar la respuesta, pero ningún modo de despliegue web vulnerable debe considerarse seguro sin el parche.

### 3. Actualizar en Debian o Ubuntu

```bash
sudo apt update
sudo apt upgrade
php --version
```

Después de instalar los paquetes corregidos, deben reiniciarse los servicios correspondientes para eliminar procesos antiguos de memoria:

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart apache2
```

Los nombres exactos dependen de la versión y de la distribución. Conviene verificar el estado del servicio y hacer una prueba funcional de la aplicación después del reinicio.

### 4. Actualizar una imagen de contenedor

No basta con reiniciar un contenedor construido a partir de una imagen antigua. Hay que obtener una imagen corregida y reconstruir el servicio:

```bash
docker pull php:8.4-fpm
docker compose build --pull --no-cache
docker compose up -d
docker compose exec php php --version
```

Se debe confirmar que la imagen resultante contiene, como mínimo, PHP 8.4.27 para esa rama.

### 5. Buscar señales de compromiso

Algunas señales que merecen investigación son:

- Procesos `php-fpm`, `php-cgi`, `apache2` o `httpd` creando shells, intérpretes o herramientas de descarga.
- Archivos PHP nuevos o modificados inesperadamente en directorios web, de subida o temporales.
- Conexiones salientes anómalas realizadas por el usuario del servidor web.
- Errores fatales, agotamientos de memoria o reinicios de workers repetidos.
- Peticiones multipartes inusualmente grandes o repetitivas.

Ejemplo de búsqueda inicial en registros locales:

```bash
journalctl --since '2026-10-20' | grep -Ei \
  'out of memory|allowed memory size|fatal error|segfault|child exited'
```

Ejemplo de revisión de archivos PHP modificados recientemente:

```bash
find /var/www -type f -name '*.php' -mtime -3 -print
```

Estos comandos únicamente generan pistas. Un resultado no confirma por sí solo un ataque y la ausencia de coincidencias tampoco demuestra que el sistema esté limpio.

## 🩹 Mitigaciones temporales

La única solución completa es instalar una versión corregida. Si no fuera posible actualizar inmediatamente, el aviso de PHP propone medidas parciales como:

- Reducir el tamaño máximo de las peticiones y limitarlo también en el proxy o servidor web.
- Desactivar administrativamente `session.upload_progress.enabled` y reiniciar todos los workers.
- En PHP 8.5, desactivar `fatal_error_backtraces` y reiniciar los workers.
- Aislar de Internet los servicios expuestos hasta poder aplicar el parche.

Estas medidas bloquean rutas conocidas, pero no cubren todas las variantes. Un WAF tampoco debe considerarse un sustituto de la actualización.

## ⛔ Qué no conviene hacer

- No probar código de explotación contra servicios públicos o sistemas ajenos.
- No asumir que WordPress, Laravel u otra aplicación actualizada compensan un intérprete PHP vulnerable.
- No limitarse a reiniciar: hay que instalar paquetes o imágenes que incluyan la corrección.
- No publicar cargas útiles funcionales mientras existan sistemas ampliamente expuestos y sin parchear.
- No declarar un servidor limpio basándose únicamente en una búsqueda rápida de registros.

## 🚨 Plan de respuesta recomendado

1. Inventariar servidores, contenedores y productos que incorporen PHP.
2. Identificar versión, proveedor y modo de ejecución.
3. Priorizar los sistemas accesibles desde Internet.
4. Actualizar a una compilación corregida y reiniciar todos los procesos PHP.
5. Verificar que la aplicación funciona y que la versión activa es la esperada.
6. Revisar registros, procesos, archivos y conexiones en busca de actividad anómala.
7. Si aparecen indicios, aislar el equipo y preservar evidencias antes de limpiarlo.
8. Rotar credenciales y secretos accesibles desde el servidor si se confirma o sospecha razonablemente un compromiso.

## ⚖️ Divulgación responsable y descargo de responsabilidad

Este documento ha sido elaborado desde la perspectiva de un **entusiasta de la ciberseguridad** con fines educativos, divulgativos y defensivos. No constituye asesoramiento profesional, legal ni una garantía de seguridad.

Las técnicas y comandos aquí descritos deben utilizarse únicamente en sistemas propios, laboratorios aislados o entornos para los que se disponga de autorización explícita. El acceso, análisis o intento de explotación de sistemas ajenos sin permiso puede ser ilegal y causar daños reales.

No se incluyen cargas útiles ni instrucciones operativas para explotar estas vulnerabilidades. Para una validación ofensiva autorizada, debe emplearse un laboratorio aislado, con instantáneas recuperables, datos ficticios y reglas de alcance claramente documentadas.

## 🔗 Fuentes

- [Aviso oficial de seguridad de PHP](https://www.php.net/archive/2026.php#2026-10-21-5)
- [CVE-2026-109618: RCE mediante corrupción por OOM](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w)
- [CVE-2026-109619: RCE mediante estado global obsoleto](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp)
- [Comunicado conjunto de ENISA sobre las vulnerabilidades](https://www.enisa.europa.eu/news/joint-statement-on-critical-php-vulnerabilities)

---

**Nota:** La información de seguridad evoluciona rápidamente. Antes de tomar decisiones operativas, deben consultarse los avisos oficiales de PHP, del sistema operativo y del proveedor de alojamiento o de las imágenes utilizadas.
