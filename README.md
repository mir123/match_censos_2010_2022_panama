# Amansando los datos del censo de Panamá de 2023

![image](https://github.com/mir123/match_censos_2010_2023_panama/assets/907400/5c7b032e-0047-414a-a788-7e9e999d991b)

Salieron los datos del censo de 2023 en Panamá, pero tienen problemas. 

1. Los códigos de lugar poblado no coinciden entre un censo y el otro
2. El Instituto Nacional de Estadística y Censo no ha publicado la cartografía correspondiente y no sabemos si lo hará

Esta es la documentación del intento de hacer encajar la mayor parte posible de los polígonos del censo del 2010, creados por la unidad cartográfica del INEC a cargo de Héctor Cedeño (Q.E.P.D.), con la lista de lugares poblados publicada por el INEC en 2023. En este repositorio incluyo los archivos y queries de PostgreSQL utilizados. Los corregimientos del INEC son muy grandes para Github pero se pueden bajar [aquí](https://nube.almanaqueazul.org/s/TefKdM6AaH5KQm9).


Saber PostgreSQL para este tipo de cosas es como un super poder. Seguro alguien que sepa más puede seguirla de aquí sin tanto trabajo manual. O quizá el INEC publicará pronto los datos geográficos.

El resultado final de este intento es un .gpkg que incluye los polígonos que coincidieron, con una columna, "pase_match" que indica en qué pase coincidieron. Los pases van más o menos bajando en cuanto a confianza del encaje, pero creo que todo son bastante confiable.

Los polígonos del 2010 que no coincidieron se incluyen con 'no encaja 2010' en la columna "pase_match". Los lugares poblados de 2023, anotados como 'no encaja 2023' son representados con un círculo proporcional a la población de 2023 más o menos alrededor del centroide del corregimiento que indica el código del lugar poblado, aunque me parece que en algunos casos está errado. La idea del círculo es facilitar un proceso manual.

## Pase 1: Asignar corregimientos del IGNTG a polígonos de cartografía del INEC censo 2010
````
CREATE TABLE lugpob_corregimientos_actualizados_2023 AS

WITH max_int AS
(SELECT lp.codigo,   MAX(ST_Area(ST_Intersection(lp.geom, c.geom))) AS max_int_area
FROM lugares_poblados_poligono_clean lp
JOIN corregimientos_pma_mbn_32617 c ON ST_Intersects(lp.geom, c.geom)	
group by lp.codigo
)

SELECT DISTINCT ON (lp.codigo)
max_int.codigo AS codigo_2010,
lp.lupo_nomb_correcto AS nombre_2010,
lp.lugpob_p_1 AS personas_2010,
'' AS codigo_2023,
'' AS nombre_2023

c.cod_correg AS correg_2023,
'' AS provincia,
'' AS distrito,

CASE WHEN
	c.cod_correg = substring(lp.codigo for 6)
	THEN 'no hay cambio'
	ELSE 'cambio'
END AS cambio,
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
ST_Area(ST_Intersection(lp.geom, c.geom)) AS max_int_area,
lp.geom

FROM lugares_poblados_poligono_clean lp
JOIN corregimientos_pma_mbn_32617 c ON ST_Intersects(lp.geom, c.geom)
JOIN max_int ON lp.codigo = max_int.codigo
LEFT JOIN panama_censo_2023_hogares_personas ON lp.codigo = panama_censo_2023_hogares_personas.codigo_9 

ORDER BY lp.codigo;
````
## Pase 2: Intentar buscar en los datos nuevos uniendo por código y coincidencia de corregimiento IGNTG 2023
````
CREATE TABLE los_que_cambiaron_tabla AS

SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
lugpob_corregimientos_actualizados_2023.geom
FROM lugpob_corregimientos_actualizados_2023
LEFT JOIN panama_censo_2023_hogares_personas ON REPLACE(unaccent(panama_censo_2023_hogares_personas.nombre),' (P)','') ~~ REPLACE(unaccent(nombre_2010),' (P)','')
AND correg_2023 = substring(panama_censo_2023_hogares_personas.codigo_9 for 6)
WHERE cambio = 'cambio' 
````	
## Pase 3: Buscar por nombre y códigos 2010 y 2023
````
CREATE TABLE los_que_cambiaron_tabla_2 AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
los_que_cambiaron_tabla.geom
FROM los_que_cambiaron_tabla
LEFT JOIN panama_censo_2023_hogares_personas ON REPLACE(unaccent(panama_censo_2023_hogares_personas.nombre),' (P)','') ~~ REPLACE(unaccent(nombre_2010),' (P)','')
AND codigo_2010 = panama_censo_2023_hogares_personas.codigo_9 
WHERE codigo_2023 IS NULL;
````
## Pase 4: A los que no coincidieron en código en la primera vuelta (probablemente error de censo en corregimiento): buscar por nombre y coincidencia de provincia y distrito
````
CREATE TABLE los_mal_calificados_tabla AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
lugpob_corregimientos_actualizados_2023.geom
FROM lugpob_corregimientos_actualizados_2023
LEFT JOIN panama_censo_2023_hogares_personas ON REPLACE(unaccent(panama_censo_2023_hogares_personas.nombre),' (P)','') ~~ REPLACE(unaccent(nombre_2010),' (P)','')
AND substring(codigo_2010 for 4) = substring(panama_censo_2023_hogares_personas.codigo_9 for 4)
WHERE cambio = 'no hay cambio' AND codigo_2010 NOT IN (SELECT panama_censo_2023_hogares_personas.codigo_9 FROM panama_censo_2023_hogares_personas)
ORDER BY codigo_2023 desc;
````
##  Pase 5: A los sobrantes del paso 4: buscar por nombre y coincidencia solo de provincia
````
CREATE TABLE los_mal_calificados_tabla2 AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,

los_mal_calificados_tabla2.geom

FROM los_mal_calificados_tabla

LEFT JOIN panama_censo_2023_hogares_personas ON REPLACE(unaccent(panama_censo_2023_hogares_personas.nombre),' (P)','') ~~ REPLACE(unaccent(nombre_2010),' (P)','')
AND substring(codigo_2010 for 2) = substring(panama_censo_2023_hogares_personas.codigo_9 for 2)
WHERE codigo_2023 IS NULL
ORDER BY codigo_2023 desc;
````
## Pase 6: A los sobrantes del paso 3: buscar por nombre y provincia 
````
CREATE TABLE los_que_cambiaron_tabla_3 AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
los_que_cambiaron_tabla_2.geom

FROM los_que_cambiaron_tabla_2

LEFT JOIN panama_censo_2023_hogares_personas ON REPLACE(unaccent(panama_censo_2023_hogares_personas.nombre),' (P)','') ~~ REPLACE(unaccent(nombre_2010),' (P)','')
AND substring(correg_2023 for 2) = substring(panama_censo_2023_hogares_personas.codigo_9 for 2) 
WHERE codigo_2023 IS NULL;
````
## Pase 7: A los sobrantes del paso 6: Buscar solo por codigo 
````
CREATE TABLE los_que_cambiaron_tabla_4 AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
los_que_cambiaron_tabla_3.geom

FROM los_que_cambiaron_tabla_3

LEFT JOIN panama_censo_2023_hogares_personas ON codigo_2010 = panama_censo_2023_hogares_personas.codigo_9 
WHERE codigo_2023 IS NULL;
````
## Pase 8: Unir a los de Panamá Oeste reemplazando 08 por 13 y usando el mismo código
````
CREATE TABLE los_que_cambiaron_tabla_5 AS
SELECT 
codigo_2010, 
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023, 
panama_censo_2023_hogares_personas.nombre AS nombre_2023, 
correg_2023, 
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
panama_censo_2023_hogares_personas.hogares as hogares_2023,
panama_censo_2023_hogares_personas.personas as personas_2023,
los_que_cambiaron_tabla_4.geom

FROM los_que_cambiaron_tabla_4

LEFT JOIN panama_censo_2023_hogares_personas ON substring(codigo_2010 from 3 for 7) = substring(panama_censo_2023_hogares_personas.codigo_9 from 3 for 7) 
WHERE codigo_2023 IS NULL AND substring(codigo_2010 for 2) = '08' AND substring(panama_censo_2023_hogares_personas.codigo_9 for 2) = '13';
````
## Crear tabla con los que encajaron, insertar a los que no encajaron del 2010 con el polígono de 2010
````
CREATE TABLE match_censos_2010_2023 AS
SELECT los_que_cambiaron_tabla.*, 'pase_2' AS cambio FROM 
los_que_cambiaron_tabla WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_que_cambiaron_tabla_2.* , 'pase_3' AS cambio FROM 
los_que_cambiaron_tabla_2 WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_mal_calificados_tabla.* , 'pase_4' AS cambio FROM 
los_mal_calificados_tabla WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_mal_calificados_tabla2.* , 'pase_5' AS cambio FROM 
los_mal_calificados_tabla2 WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_que_cambiaron_tabla_3.* , 'pase_6' AS cambio FROM 
los_que_cambiaron_tabla_3 WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_que_cambiaron_tabla_4.* , 'pase_7' AS cambio FROM 
los_que_cambiaron_tabla_4 WHERE codigo_2023 IS NOT NULL
UNION ALL
SELECT los_que_cambiaron_tabla_5.* , 'pase_8' AS cambio FROM 
los_que_cambiaron_tabla_5;

INSERT INTO match_censos_2010_2023 (
codigo_2010,
nombre_2010,
personas_2010,
codigo_2023,
nombre_2023,
correg_2023,
provincia, 
distrito, 
hogares_2023,
personas_2023,
geom,
cambio
)
SELECT 
codigo_2010,
nombre_2010,
personas_2010,
panama_censo_2023_hogares_personas.codigo_9 AS codigo_2023,
panama_censo_2023_hogares_personas.nombre AS nombre_2023,
correg_2023,
substring(correg_2023 for 2) AS provincia, 
substring(correg_2023 from 3 for 2) AS distrito, 
hogares_2023,
personas_2023,
geom,
'pase_1' AS cambio
FROM lugpob_corregimientos_actualizados_2023
LEFT JOIN panama_censo_2023_hogares_personas ON codigo_2010 = panama_censo_2023_hogares_personas.codigo_9 
WHERE lugpob_corregimientos_actualizados_2023.cambio = 'no hay cambio' AND panama_censo_2023_hogares_personas.codigo_9 IS NOT NULL;
````
## Insertar a los que no encajaron del 2010 con su polígono
````
INSERT INTO match_censos_2010_2023 (
codigo_2010,
nombre_2010,
personas_2010,
codigo_2023,
nombre_2023,
correg_2023,
provincia, 
distrito, 
hogares_2023,
personas_2023,
geom,
cambio
)
SELECT 
lp.codigo AS codigo_2010,
lp.lupo_nomb_correcto AS nombre_2010,
lp.lugpob_p_1 AS personas_2010,
'' AS codigo_2023,
'' AS nombre_2023,
lugpob_corregimientos_actualizados_2023.correg_2023,
'' AS provincia,
'' AS distrito,
NULL AS hogares_2023,
NULL AS personas_2023,
lp.geom,
'no encaja 2010' AS cambio
FROM lugares_poblados_poligono_clean lp
JOIN lugpob_corregimientos_actualizados_2023 ON lugpob_corregimientos_actualizados_2023.codigo_2010 = lp.codigo
WHERE lp.codigo NOT IN (SELECT codigo_2010 FROM match_censos_2010_2023);
````
# Insertar a los que no encajaron de 2023 y crear un circulo proporcional a la poblacion en el centroide del corregimiento (con una traslación aleatoria)
````
INSERT INTO match_censos_2010_2023 (
codigo_2010,
nombre_2010,
personas_2010,
codigo_2023,
nombre_2023,
correg_2023,
provincia, 
distrito, 
hogares_2023,
personas_2023,
geom,
cambio
)

SELECT 
'' AS codigo_2010,
'' AS nombre_2010,
NULL AS personas_2010,
codigo_9 AS codigo_2023,
nombre AS nombre_2023,
substring(codigo_9 for 6) AS correg_2023,
provincia AS provincia,
distrito AS distrito,
hogares AS hogares_2023,
personas AS personas_2023,
ST_Multi(ST_Buffer(ST_Translate(
	ST_Centroid(corregimientos_pma_mbn_32617.geom)
	,random()*1000
	,random()*1000
)
		 , (personas*500/3.14)^0.5)) AS geom,
'no encaja 2023' AS cambio
FROM panama_censo_2023_hogares_personas
LEFT JOIN corregimientos_pma_mbn_32617 ON corregimientos_pma_mbn_32617.cod_correg = substring(codigo_9 for 6)
WHERE codigo_9 NOT IN (SELECT codigo_2023 FROM match_censos_2010_2023)
ORDER BY personas_2023 DESC;
````
