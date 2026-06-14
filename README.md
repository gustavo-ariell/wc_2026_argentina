# Consultas SQL — Mundial FIFA 2026 · Argentina (Grupo J)

> Base de datos: `mundial_2026_argentina`
> Motor: MariaDB / MySQL 8.0+
> Ordenadas de menor a mayor complejidad.

---

## Nivel 1 — Consultas simples (SELECT, WHERE, ORDER BY)

### 1. Listar todos los convocados por dorsal



```bash
SELECT dorsal, nombre, posicion, ciudad_origen, club
  FROM futbolistas
 ORDER BY dorsal;

```

### 2. Filtrar jugadores por posicion

```bash
SELECT dorsal, nombre, edad, club
  FROM futbolistas
 WHERE posicion = 'Delantero'
 ORDER BY dorsal;

```

### 3. Jugadores nacidos en una provincia especifica

```bash
SELECT nombre, posicion, ciudad_origen
  FROM futbolistas
 WHERE ciudad_origen LIKE '%Cordoba%'
 ORDER BY nombre;

```

### 2. Filtrar jugadores por posicion

```bash
SELECT dorsal, nombre, edad, club
  FROM futbolistas
 WHERE posicion = 'Delantero'
 ORDER BY dorsal;

```

### 3. Jugadores nacidos en una provincia especifica

```bash
SELECT nombre, posicion, ciudad_origen
  FROM futbolistas
 WHERE ciudad_origen LIKE '%Cordoba%'
 ORDER BY nombre;

```

---
## Nivel 2 - Agregaciones (GROUP BY, COUNT, SUM)

### 4. Cantidad de convocados por club

```bash
SELECT club, COUNT(*) AS cantidad
  FROM futbolistas
 GROUP BY club
 ORDER BY cantidad DESC;

```

### 5. Goles totales por jugador en la fase de grupos

```bash
SELECT f.dorsal,
       f.nombre,
       SUM(fp.goles)           AS total_goles,
       SUM(fp.asistencias)     AS total_asistencias,
       SUM(fp.minutos_jugados) AS minutos
  FROM futbolistas f
  JOIN futbolista_partido fp ON f.id_futbolista = fp.id_futbolista
 GROUP BY f.id_futbolista
 ORDER BY total_goles DESC, total_asistencias DESC;

```

### 6. Total de goles de Argentina por partido

```bash
SELECT p.fecha,
       CONCAT('Argentina vs ', p.rival) AS partido,
       SUM(fp.goles) AS goles_argentina
  FROM partidos p
  JOIN futbolista_partido fp ON p.id_partido = fp.id_partido
 GROUP BY p.id_partido
 ORDER BY p.fecha;

```
---
## Nivel 3 - JOIN entre multiples tablas

### 7. Jugadores con su tecnico asignado


```bash
SELECT f.dorsal,
       f.nombre   AS jugador,
       f.posicion,
       ct.nombre  AS tecnico,
       ct.cargo
  FROM futbolistas f
  JOIN cuerpo_tecnico ct ON f.id_tecnico = ct.id_tecnico
 ORDER BY ct.nombre, f.dorsal;

```

### 8. Goleadores con su asistidor por partido


```bash
SELECT p.fecha,
       CONCAT('vs ', p.rival)   AS rival,
       goleador.nombre          AS goleador,
       fp.goles,
       asistidor.nombre         AS asistidor,
       fp2.asistencias
  FROM futbolista_partido fp
  JOIN futbolistas goleador   ON fp.id_futbolista = goleador.id_futbolista
  JOIN partidos p             ON fp.id_partido     = p.id_partido
  LEFT JOIN futbolista_partido fp2
       ON fp2.id_partido    = fp.id_partido
       AND fp2.asistencias > 0
  LEFT JOIN futbolistas asistidor
       ON fp2.id_futbolista = asistidor.id_futbolista
 WHERE fp.goles > 0
 ORDER BY p.fecha, fp.goles DESC;

```
## Nivel 4 - Subconsultas


### 9. Jugadores que marcaron al menos un gol

```bash
SELECT dorsal, nombre, posicion, club
  FROM futbolistas
 WHERE id_futbolista IN (
       SELECT DISTINCT id_futbolista
         FROM futbolista_partido
        WHERE goles > 0
 )
 ORDER BY nombre;

```

### 10. Jugadores por encima del promedio de goles


```bash
SELECT f.dorsal,
       f.nombre,
       SUM(fp.goles) AS total_goles
  FROM futbolistas f
  JOIN futbolista_partido fp ON f.id_futbolista = fp.id_futbolista
 GROUP BY f.id_futbolista
HAVING total_goles > (
       SELECT AVG(sub.total)
         FROM (
               SELECT SUM(fp2.goles) AS total
                 FROM futbolista_partido fp2
                GROUP BY fp2.id_futbolista
              ) sub
 )
 ORDER BY total_goles DESC;

```

### 11. Jugadores que fueron titulares en los tres partidos

```bash
SELECT f.dorsal, f.nombre, f.posicion
  FROM futbolistas f
 WHERE 3 = (
       SELECT COUNT(*)
         FROM futbolista_partido fp
        WHERE fp.id_futbolista = f.id_futbolista
          AND fp.titular = 1
 )
 ORDER BY f.dorsal;


```
## Nivel 5 - HAVING, JOIN avanzado, funciones de texto

### 12. Clubes con mas de 2 jugadores convocados

```bash
SELECT club, COUNT(*) AS convocados
  FROM futbolistas
 GROUP BY club
HAVING convocados > 2
 ORDER BY convocados DESC;

```

### 13. Resumen completo de cada partido con estadio y fase

```bash
SELECT p.fecha,
       p.estadio,
       p.ciudad,
       CONCAT('Argentina vs ', p.rival) AS partido,
       SUM(fp.goles)                    AS goles_arg,
       COUNT(DISTINCT fp.id_futbolista) AS jugadores_utilizados
  FROM partidos p
  JOIN futbolista_partido fp ON p.id_partido = fp.id_partido
 GROUP BY p.id_partido
 ORDER BY p.fecha;


```

### 14. Provincia con mas convocados

```bash
SELECT SUBSTRING_INDEX(ciudad_origen, ',', -1) AS provincia,
       COUNT(*) AS convocados
  FROM futbolistas
 GROUP BY provincia
 ORDER BY convocados DESC;


```
---
## Nivel 6 - Window functions y CTE


### 15. Ranking de goleadores con RANK()

```bash
SELECT RANK() OVER (ORDER BY SUM(fp.goles) DESC) AS pos,
       f.dorsal,
       f.nombre,
       SUM(fp.goles)       AS goles,
       SUM(fp.asistencias) AS asistencias
  FROM futbolistas f
  JOIN futbolista_partido fp ON f.id_futbolista = fp.id_futbolista
 GROUP BY f.id_futbolista
 ORDER BY pos;

```

### 16. Acumulado de goles de Messi partido a partido

```bash
WITH goles_messi AS (
    SELECT p.fecha,
           CONCAT('vs ', p.rival) AS rival,
           fp.goles,
           fp.asistencias,
           fp.minutos_jugados
      FROM futbolista_partido fp
      JOIN partidos p ON fp.id_partido = p.id_partido
     WHERE fp.id_futbolista = 10
     ORDER BY p.fecha
)
SELECT fecha,
       rival,
       goles,
       asistencias,
       SUM(goles)       OVER (ORDER BY fecha) AS goles_acumulados,
       SUM(asistencias) OVER (ORDER BY fecha) AS asist_acumuladas
  FROM goles_messi;

```

### 17. Porcentaje de minutos jugados por jugador

```bash
WITH minutos_totales AS (
    SELECT f.id_futbolista,
           f.dorsal,
           f.nombre,
           SUM(fp.minutos_jugados) AS minutos,
           SUM(fp.titular)         AS partidos_titular,
           COUNT(fp.id_partido)    AS partidos_jugados
      FROM futbolistas f
      JOIN futbolista_partido fp ON f.id_futbolista = fp.id_futbolista
     GROUP BY f.id_futbolista
)
SELECT dorsal,
       nombre,
       partidos_jugados,
       partidos_titular,
       minutos,
       ROUND(minutos / (partidos_jugados * 90) * 100, 1) AS pct_minutos
  FROM minutos_totales
 ORDER BY pct_minutos DESC;


```
--- 
## Nivel 7 - Consultas avanzadas (CTE multiple + Window + Multi-JOIN)


### 18. Informe completo de rendimiento ofensivo con ranking acumulado

```bash
WITH rendimiento AS (
    SELECT fp.id_futbolista,
           fp.id_partido,
           fp.goles,
           fp.asistencias,
           fp.minutos_jugados,
           fp.titular,
           p.fecha,
           CONCAT('vs ', p.rival) AS rival
      FROM futbolista_partido fp
      JOIN partidos p ON fp.id_partido = p.id_partido
),
ranking AS (
    SELECT r.id_futbolista,
           f.dorsal,
           f.nombre,
           f.posicion,
           f.ciudad_origen,
           f.club,
           ct.nombre AS tecnico,
           r.fecha,
           r.rival,
           r.goles,
           r.asistencias,
           r.minutos_jugados,
           r.titular,
           SUM(r.goles)       OVER (PARTITION BY r.id_futbolista ORDER BY r.fecha) AS goles_acum,
           SUM(r.asistencias) OVER (PARTITION BY r.id_futbolista ORDER BY r.fecha) AS asist_acum,
           SUM(r.goles)       OVER () AS total_goles_torneo,
           RANK() OVER (ORDER BY SUM(r.goles) OVER (PARTITION BY r.id_futbolista) DESC) AS ranking_goleador
      FROM rendimiento r
      JOIN futbolistas f     ON r.id_futbolista = f.id_futbolista
      JOIN cuerpo_tecnico ct ON f.id_tecnico    = ct.id_tecnico
)
SELECT ranking_goleador,
       dorsal,
       nombre,
       posicion,
       ciudad_origen,
       club,
       tecnico,
       fecha,
       rival,
       goles,
       asistencias,
       minutos_jugados,
       CASE titular WHEN 1 THEN 'Si' ELSE 'No' END AS titular,
       goles_acum,
       asist_acum
  FROM ranking
 WHERE goles > 0 OR asistencias > 0
 ORDER BY ranking_goleador, fecha;

```
### 19. Comparativa de rendimiento por linea de juego

```bash
WITH lineas AS (
    SELECT
        CASE
            WHEN f.posicion = 'Arquero'       THEN 'Arqueros'
            WHEN f.posicion = 'Defensor'      THEN 'Defensores'
            WHEN f.posicion = 'Mediocampista' THEN 'Mediocampistas'
            WHEN f.posicion = 'Delantero'     THEN 'Delanteros'
        END AS linea,
        f.nombre,
        SUM(fp.goles)           AS goles,
        SUM(fp.asistencias)     AS asist,
        SUM(fp.minutos_jugados) AS minutos,
        COUNT(fp.id_partido)    AS partidos
      FROM futbolistas f
      JOIN futbolista_partido fp ON f.id_futbolista = fp.id_futbolista
     GROUP BY f.id_futbolista
)
SELECT linea,
       COUNT(*)                             AS jugadores,
       SUM(goles)                           AS total_goles,
       SUM(asist)                           AS total_asist,
       ROUND(AVG(minutos), 0)              AS prom_minutos,
       ROUND(SUM(goles) / COUNT(*), 2)     AS goles_x_jugador
  FROM lineas
 GROUP BY linea
 ORDER BY total_goles DESC;


```
### 20. Informe completo de la fase de grupos

```bash
WITH goles_por_partido AS (
    SELECT fp.id_partido,
           SUM(fp.goles) AS total_goles
      FROM futbolista_partido fp
     GROUP BY fp.id_partido
),
info_partidos AS (
    SELECT p.id_partido,
           p.fecha,
           CONCAT('Argentina vs ', p.rival) AS partido,
           p.rival,
           p.estadio,
           p.ciudad,
           p.fase,
           gpp.total_goles AS goles_arg
      FROM partidos p
      JOIN goles_por_partido gpp ON p.id_partido = gpp.id_partido
)
SELECT ip.fecha,
       ip.partido,
       ip.goles_arg,
       ip.estadio,
       ip.ciudad,
       ip.fase,
       GROUP_CONCAT(
           CONCAT(g.nombre, ' (', fp.goles, ')')
           ORDER BY fp.goles DESC
           SEPARATOR ', '
       ) AS goleadores
  FROM info_partidos ip
  JOIN futbolista_partido fp ON ip.id_partido    = fp.id_partido
  JOIN futbolistas g         ON fp.id_futbolista = g.id_futbolista
 WHERE fp.goles > 0
 GROUP BY ip.id_partido
 ORDER BY ip.fecha;

```
