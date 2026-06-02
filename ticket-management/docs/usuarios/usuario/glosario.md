---
rol: usuario
tipo: glosario
audiencia: usuario_final
fuente: ticket-management
---

# Glosario — Usuario

Definiciones de todos los términos que aparecen en el sistema de tickets.

---

**Ticket**
Registro formal de una solicitud, problema o tarea dentro del sistema. Cada ticket tiene un identificador único, un estado, y un responsable asignado. Es la unidad básica de trabajo del sistema.

**Estado**
Indica en qué etapa del proceso se encuentra un ticket. Los cuatro estados posibles son: Abierto, En Progreso, En Revisión y Cerrado. Solo el equipo (Manager o Admin) puede cambiar el estado.

**Abierto (OPEN)**
Estado inicial de todo ticket recién creado. Significa que está registrado y esperando ser atendido.

**En Progreso (IN_PROGRESS)**
El equipo está trabajando activamente en el ticket. Ya tiene una persona asignada.

**En Revisión (IN_REVIEW)**
El trabajo está terminado o casi terminado y se está verificando antes del cierre. Puede requerir tu confirmación.

**Cerrado (CLOSED)**
Estado final e irreversible. El ticket fue resuelto. No se puede reabrir — si el problema persiste, se crea un ticket nuevo.

**Prioridad**
Nivel de urgencia del ticket. Tiene cuatro niveles:
- `Baja`: no urgente, puede esperar
- `Media`: necesita atención en tiempo razonable
- `Alta`: afecta el trabajo, necesita atención pronta
- `Crítica`: bloqueante total, atención inmediata requerida

**Tipo de ticket**
Clasificación del tipo de solicitud:
- `Tarea (TASK)`: algo que necesitas que se haga
- `Bug (BUG)`: algo que no funciona correctamente
- `Feature (FEATURE)`: solicitud de mejora o nueva funcionalidad
- `Documentación (DOCS)`: solicitud de información o documentación

**Asignado a**
Persona del equipo responsable de resolver el ticket. Un ticket puede no tener asignación inicial y ser asignado después.

**Creado por**
Usuario que abrió el ticket. En tu caso, siempre serás tú.

**Solicitante (Requester)**
Persona que originó la solicitud. Cuando creas un ticket directamente, eres tú. Cuando un ticket llega por correo electrónico, el solicitante es quien envió el correo.

**Comentario**
Mensaje agregado al ticket por cualquier participante. Sirve para comunicarse, dar información adicional o hacer seguimiento. Todos los comentarios quedan registrados con fecha y autor.

**Adjunto**
Archivo (imagen, documento, captura de pantalla) vinculado a un ticket. Los adjuntos tienen un límite de 10 MB por archivo. Como Usuario puedes ver y descargar adjuntos pero no subirlos.

**Historial / Eventos**
Registro cronológico de todos los cambios que tuvo el ticket: cambios de estado, reasignaciones, modificaciones de prioridad. Cada entrada muestra quién hizo el cambio y cuándo.

**Clasificación**
Dominio al que pertenece el ticket. Las clasificaciones disponibles son:
- `IT`: tecnología e infraestructura (hardware, software, red, acceso, correo, impresora, VPN, servidor, base de datos, seguridad)
- `Cliente`: atención y soporte al cliente (facturación, reclamo, consulta, devolución, garantía, soporte, pedido, envío)
- `Operaciones`: procesos y logística (logística, compras, inventario, mantenimiento, producción, calidad, proveedores)
- `Otro`: sin clasificación específica (general, sin clasificar)

**Categoría**
Subcategoría dentro de la Clasificación. Por ejemplo, dentro de IT puedes tener categorías como "hardware", "red" o "acceso".

**Fecha límite (Due Date)**
Fecha en la que necesitas que el ticket esté resuelto. Es opcional al crear el ticket. El equipo la usa para priorizar el trabajo.

**Horas estimadas**
Estimación del tiempo que tomará resolver el ticket. Lo define el equipo, no el usuario.

**Horas reales**
Tiempo efectivo que tomó resolver el ticket. Lo registra el equipo al cerrar.

**Borrador (Draft)**
Estado especial de un ticket que aún no entró al flujo normal. Los borradores son creados automáticamente cuando llega un ticket por correo electrónico y necesita revisión y clasificación antes de activarse. Como Usuario, no ves los borradores directamente.

**Archivado**
Un ticket archivado es uno que fue ocultado de las vistas normales. Solo el Admin puede archivar tickets. El archivado es diferente al cierre — archivar es para limpieza de la lista, cerrar es para resolver el caso.

**Rol**
Nivel de acceso que tienes en el sistema. Los tres roles son:
- `Usuario`: acceso básico, puede crear y consultar tickets
- `Manager`: acceso intermedio, puede gestionar tickets del equipo
- `Admin`: acceso total, puede configurar el sistema

**Notificación**
Aviso automático que el sistema envía cuando ocurre un evento relevante en tu ticket (por ejemplo, cuando cambia a un estado específico). La configuración de notificaciones la controla el Admin.
