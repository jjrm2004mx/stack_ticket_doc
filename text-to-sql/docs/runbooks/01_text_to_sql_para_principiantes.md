# 01 — text-to-sql para principiantes

## En una frase

Es el servicio que convierte una pregunta como "¿cuántos tickets abiertos tiene Nómina este mes?" en
una consulta SQL segura, y que después convierte el resultado en una frase. **No guarda nada y nunca
se conecta a una base de datos.**

## Quién hace qué

```
Usuario ─► Portal ─► ticket-management (TM) ─────────────► text-to-sql (este motor)
                       │  "esta es la pregunta, estos los       │
                       │   nombres de mis estados y áreas"      │
                       │ ◄──── SQL con un hueco para el cliente ┘
                       │
                       ├─► base de TM: TM llena el hueco con el cliente de la sesión
                       │   y ejecuta con un usuario de base que solo puede leer
                       │
                       ├─► text-to-sql: "redacta la respuesta con estas filas"
                       ▼
                     Portal: texto + tabla
```

| Pieza | Qué hace | Qué NO hace |
|---|---|---|
| **text-to-sql** | Sabe qué significa cada vista y columna (el **catálogo**), escribe el SQL con ayuda del modelo de lenguaje, lo revisa regla por regla y le agrega el filtro del cliente | No ejecuta el SQL, no tiene contraseñas de bases, no sabe qué cliente pregunta, no guarda preguntas |
| **ticket-management** | Decide quién puede preguntar, pone el cliente, ejecuta, registra cada pregunta | No sabe escribir SQL ni conoce el catálogo |
| **El modelo** (OCI) | Propone el SQL y redacta la frase | Nunca toca la base; lo que propone pasa por el validador |

## Las tres respuestas posibles a una pregunta

| Estado | Cuándo | Ejemplo |
|---|---|---|
| `SQL` | Se pudo escribir una consulta válida | "¿Cuántos tickets abiertos hay?" |
| `CLARIFY` | Hay que preguntar algo antes | "¿Cuántos tickets tiene Ana?" con dos Anas |
| `OUT_OF_SCOPE` | No se puede o no se debe | "Borra los tickets cancelados", "¿cuántos vencidos hay?" |
| `INVALID` | El modelo no logró un SQL que pase las reglas en dos intentos | Rara vez; se pide reformular |

## Por qué es seguro aunque el modelo se equivoque

1. **Lista blanca:** solo se acepta una consulta de lectura, solo con las vistas del catálogo, solo
   con funciones conocidas. Todo lo demás se rechaza (escrituras, `set_config`, `pg_sleep`, tablas
   como `users`, nombres con esquema…).
2. **Filtro obligatorio:** el motor reescribe cada consulta para que cada vista pase primero por
   `WHERE tenant_id = ?`. El `?` lo llena TM con el cliente de la sesión; el motor nunca lo conoce.
3. **El SQL que sale lo escribe el motor**, desde lo que validó, no el texto del modelo. Antes de
   devolverlo lo vuelve a leer; si algo no cuadra, no lo devuelve.
4. **La base de TM tiene sus propias barreras** (usuario de solo lectura, seguridad por fila): aunque
   el motor fallara, no entregaría filas de otro cliente.

## El catálogo

Una carpeta por aplicación en `catalogs/`. La de TM (`catalogs/ticket-management/`) describe las ocho
vistas del esquema `insights`: tickets, cambios de estado, estados, áreas, categorías, personas,
borradores y archivados. Trae las reglas ("abierto" es `is_open`, "este mes" es el mes calendario…) y
16 ejemplos pregunta→SQL. Ver `docs/catalogo.md`.

## Dónde vive y cómo se ve que está vivo

- En el servidor: `~/stack_ticket/text-to-sql`, contenedor `text-to-sql`, puerto `127.0.0.1:8003`.
- `curl -s http://localhost:8003/health` → `{"status":"ok", "provider":"oci", …, "catalogs":[…]}`.
- Lo arranca y vigila `infra-monitoring` (como sexto proyecto de `startup.sh`, opcional).

## Glosario

| Palabra | Qué es |
|---|---|
| Catálogo | La descripción de las vistas de una aplicación que el motor conoce |
| Objeto | Una vista del catálogo (`tickets`, `states`…) |
| Filtro obligatorio | La condición que el motor agrega a cada objeto (`tenant_id = ?`) |
| Parámetro sin valor | El `?` que llena la aplicación |
| Llave | `X-Service-Key`, una por aplicación |
| `usageKey` | Una etiqueta opaca que manda la aplicación para repartir su cupo (en TM, un cliente) |
| Proveedor | De dónde sale el modelo: `oci` (Rocky) u `ollama` (local) |
