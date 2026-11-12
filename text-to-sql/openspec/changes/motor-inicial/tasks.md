> Las secciones siguen las etapas del plan versión 4 (`docs/tecnico/PLAN_PREGUNTAS_SOBRE_LOS_DATOS.md`). No se escribe código sin hablarlo antes con el dueño, y antes de cada etapa se avisa el esfuerzo del modelo. Este cambio vive en `stack_ticket_doc/text-to-sql/openspec/` hasta que exista el repo.

## 0. Antes de programar

- [x] 0.1 Visto bueno del dueño a esta propuesta y a sus hermanas (`preguntas-sobre-los-datos` en ticket-management, `motor-en-el-ecosistema` en infra-monitoring)
- [ ] 0.2 Confirmar el nombre del repo (`text-to-sql` es provisional) y crearlo
- [ ] 0.3 Esperar el resultado de la compuerta 1 de ticket-management: si la base no garantiza el aislamiento, el motor no se construye

## 1. Repo y plantilla (etapa 4)

- [ ] 1.1 Repo con `CLAUDE.md`, `README.md`, `.env.example`, `NOTICE` (reglas portadas de Wren, Apache-2.0), skills de OpenSpec en `.claude/skills` y este cambio movido a `openspec/changes/motor-inicial/`
- [ ] 1.2 `Dockerfile`, `docker-compose.yml` (contenedor `text-to-sql`, `127.0.0.1:8003`, solo `ticket-management-network`, `~/.oci` de solo lectura, journald, chequeo de salud) y `start.sh`
- [ ] 1.3 Dependencias fijadas: FastAPI, SQLGlot, `langchain-oci`, `langchain-ollama`, `prometheus-fastapi-instrumentator`; ningún controlador de base de datos

## 2. Catálogo (etapa 4)

- [ ] 2.1 Cargador y validación del catálogo al arrancar (formato, filtros obligatorios, referencias, ejemplos que pasan el validador); documento `docs/catalogo.md`
- [ ] 2.2 Catálogo `ticket-management` 1.0: ocho objetos con sus columnas (`ETAPA_1_DEFINICION_DE_VISTAS.md` §3), `tenant_id` oculto, conjuntos de valores, valores fijos con etiquetas, reglas, 16 ejemplos y 7 preguntas negativas
- [ ] 2.3 `contract.json` generado y su prueba; `GET /v1/catalogs/{application}`
- [ ] 2.4 Revisar el catálogo contra la lista de huecos semánticos de Wren (`gap_catalog.md`)

## 3. Validador y filtro (etapa 4, esfuerzo extra alto)

- [ ] 3.1 Validador: las 15 reglas con sus códigos, la lista blanca de funciones, operadores, tipos y literales
- [ ] 3.2 Filtro obligatorio por CTE, columnas ocultas, objetos compartidos, parámetros `named` y `qmark`
- [ ] 3.3 SQL final generado desde el árbol, límite de filas y comprobación de relectura (falla cerrado)
- [ ] 3.4 `POST /v1/validate`
- [ ] 3.5 Pruebas de todos los casos negativos de `ETAPA_2_DISENO.md` §8 y de los 16 ejemplos, sin modelo

## 4. Modelo y rutas (etapa 4)

- [ ] 4.1 Adaptadores copiados tal cual de `ticket-agent/src/adapters/` con nota de origen; `PROVIDER`, tiempos y `modelCalls`
- [ ] 4.2 Prompt de SQL (catálogo permitido, objetos no disponibles, valores, reglas, ejemplos, secciones de datos) y salida estructurada; reglas de redacción de SQL de Wren como base
- [ ] 4.3 `POST /v1/sql`: tubería fija con un reintento, `previous`, `dbError`, `CLARIFY`, `OUT_OF_SCOPE`, versión del catálogo
- [ ] 4.4 `POST /v1/answer`: límites de entrada, texto plano, reglas de redacción del catálogo
- [ ] 4.5 Llaves por aplicación y límites por aplicación y por clave de uso
- [ ] 4.6 `/health`, `/metrics` con las métricas propias, identificador de petición y logs sin contenido
- [ ] 4.7 Pruebas con proveedor simulado: reintento, dos fallos, aclaración, fuera de alcance, error y tiempo agotado del proveedor, llaves, límites, logs sin preguntas

## 5. Operación (etapa 4)

- [ ] 5.1 Runbooks 01 (para principiantes), 02 (arquitectura técnica) y 03 (guía operacional) y su copia en `stack_ticket_doc/text-to-sql/`
- [ ] 5.2 Correr en local con `PROVIDER=ollama` o simulado; suite completa
- [ ] 5.3 Con permiso del dueño: limpiar imágenes viejas en Rocky y levantar el motor ahí con `PROVIDER=oci`; una muestra mínima de preguntas contra `/v1/sql` (poco consumo de OCI)

## 6. Compuerta 2 (etapa 6, junto con ticket-management)

- [ ] 6.1 Casos negativos con el modelo (N1 a N7, instrucciones en un estado y en un título)
- [ ] 6.2 El filtro a solas: SQL final de las preguntas de referencia en una base de prueba con la seguridad por fila apagada
- [ ] 6.3 Muestra de las preguntas de referencia: exactitud y tiempo por llamada; ajustar reglas, ejemplos y tiempos
- [ ] 6.4 Decidir con lo medido si `string_agg` y los percentiles siguen en la lista blanca

## 7. Cierre (etapa 9)

- [ ] 7.1 Guía de adopción, ejecutor de ejemplo y paquete de pruebas para la base de la aplicación
- [ ] 7.2 Corrida completa de las preguntas de referencia
- [ ] 7.3 Simulación de archivado y archivar, junto con los cambios hermanos
