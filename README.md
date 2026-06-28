# Guía de Defensa: Inyección SQL en Plataformas de E-commerce

> **Propósito:** Documento de referencia defensiva para entender cómo ocurre un ataque de inyección SQL (SQLi), cómo detectarlo a tiempo y cómo responder. Pensado para equipos de desarrollo, DevOps y administradores de tiendas online.

---

## 📌 Tabla de Contenidos

1. [¿Qué es la inyección SQL?](#qué-es-la-inyección-sql)
2. [Anatomía del ataque paso a paso](#anatomía-del-ataque-paso-a-paso)
3. [Puntos de entrada típicos en e-commerce](#puntos-de-entrada-típicos-en-e-commerce)
4. [Señales de compromiso](#señales-de-compromiso)
5. [Plan de mitigación inmediata](#plan-de-mitigación-inmediata)
6. [Prevención a largo plazo](#prevención-a-largo-plazo)
7. [Checklist rápido](#checklist-rápido)

---

![Guía de defensa: inyección SQL en e-commerce — infografía completa]<img width="2800" height="2724" alt="infografia-sql-injection" src="https://github.com/user-attachments/assets/340ec9fb-2661-4c35-9d7b-1906b8db1868" />


## ¿Qué es la inyección SQL?

La inyección SQL ocurre cuando una aplicación construye consultas a la base de datos concatenando directamente datos que vienen del usuario (formularios, parámetros de URL, cookies, headers) sin separarlos correctamente del código SQL. Si el atacante logra que parte de su input sea interpretado como **comandos** en lugar de como **datos**, puede alterar la lógica de la consulta original.

En una tienda online esto es especialmente crítico porque la base de datos suele contener: credenciales de clientes, direcciones, historial de compras, tokens de sesión y, en integraciones mal diseñadas, datos de pago.

---

## Anatomía del ataque paso a paso

```
Reconocimiento → Detección del punto vulnerable → Explotación → Extracción/Persistencia → Limpieza de huellas
```

### 1. Reconocimiento
El atacante identifica campos que interactúan con la base de datos: búsqueda de productos, login, filtros de categoría, parámetros `id` en URLs de producto, campos de cupones, etc. Usa herramientas automatizadas o pruebas manuales con caracteres especiales (`'`, `"`, `--`, `;`).

### 2. Detección del punto vulnerable
Prueba entradas que alteran la sintaxis SQL (por ejemplo, un apóstrofe simple en un campo de búsqueda) y observa la respuesta del servidor. Un error de base de datos expuesto en pantalla, un cambio de comportamiento, o una demora distinta en la respuesta (técnica *time-based*) confirma que el parámetro no está bien saneado.

### 3. Explotación
Dependiendo de cómo responda la aplicación, el atacante elige una técnica:
- **Basada en errores:** provoca mensajes de error que revelan la estructura de la base de datos.
- **Basada en UNION:** combina su propia consulta con la original para extraer datos de otras tablas (por ejemplo, la tabla de usuarios).
- **Booleana ciega:** hace preguntas verdadero/falso a través de cambios sutiles en la respuesta cuando no hay mensajes de error visibles.
- **Basada en tiempo:** mide retrasos provocados deliberadamente para inferir información bit a bit cuando no hay ninguna otra señal observable.

### 4. Extracción o persistencia
Si la explotación tiene éxito, el atacante puede leer tablas completas (usuarios, pedidos, tokens), modificar registros (cambiar precios, saldos, roles de administrador) o, en bases de datos con privilegios elevados, intentar ejecutar comandos a nivel de sistema operativo para obtener un punto de apoyo más amplio.

### 5. Limpieza de huellas
Atacantes más sofisticados intentan borrar o alterar entradas de logs relacionadas con sus consultas para dificultar la investigación posterior.

---

## Puntos de entrada típicos en e-commerce

| Componente | Riesgo típico |
|---|---|
| Barra de búsqueda de productos | Consultas dinámicas construidas con el texto buscado |
| Filtros y parámetros de URL (`?id=`, `?categoria=`) | IDs numéricos sin validar tipo/rango |
| Formularios de login / registro | Validación de credenciales mal parametrizada |
| Campos de cupones o códigos promocionales | Lógica de descuento con concatenación directa |
| Integraciones de terceros (reseñas, chat, CRM) | Datos externos insertados sin sanear |
| Paneles de administración expuestos | Menor escrutinio de seguridad que el frontend público |

---

## Señales de compromiso

Presta atención a estos indicadores, idealmente correlacionados entre sí (uno solo puede ser ruido normal):

- **Errores de base de datos visibles al usuario** en páginas públicas (mensajes con nombres de tablas, columnas o fragmentos de SQL).
- **Picos de tráfico anómalo** hacia un mismo endpoint, con parámetros que contienen comillas, `UNION`, `SELECT`, `--`, `OR 1=1`, codificación URL sospechosa, etc.
- **Solicitudes con tiempos de respuesta inusualmente largos** y repetidos hacia el mismo parámetro (posible técnica basada en tiempo).
- **Aumento repentino de intentos de login fallidos** combinados con parámetros no estándar en el campo de usuario/contraseña.
- **Cambios inexplicados de datos:** precios alterados, cuentas con privilegios de administrador creadas sin registro de alta normal, pedidos modificados retroactivamente.
- **Tráfico saliente de la base de datos hacia destinos no habituales**, o consultas que tardan más de lo normal en el motor de base de datos.
- **Nuevas cuentas de usuario en la base de datos** que no pasaron por el flujo normal de registro (sin verificación de email, sin historial de carrito, etc.).
- **Alertas del WAF (Web Application Firewall)** marcando patrones de inyección, si ya tienes uno desplegado.
- **Entradas de log truncadas, eliminadas o con huecos sospechosos** en el rango de tiempo del incidente.

---

## Plan de mitigación inmediata

Si se detecta o sospecha un ataque activo, sigue este orden:

1. **Contener sin destruir evidencia.** No apagues el servidor de inmediato si necesitas preservar logs en memoria; en su lugar, bloquea las IPs/origen sospechoso a nivel de firewall o WAF y considera poner el sitio en modo mantenimiento si el riesgo es alto.
2. **Revocar credenciales y sesiones.** Rota inmediatamente las contraseñas de la base de datos, claves de API y secretos de sesión. Invalida todos los tokens de sesión activos para forzar nuevos logins.
3. **Aislar la base de datos comprometida.** Si es posible, separa temporalmente la base de datos de producción de la red pública y trabaja sobre una réplica o backup para el análisis forense.
4. **Parchear el punto de entrada explotado.** Identifica el endpoint o parámetro específico usado en el ataque (revisando logs del servidor web y de la base de datos) y aplica una corrección de emergencia: parametrizar la consulta, agregar validación estricta de tipo, o deshabilitar temporalmente la funcionalidad afectada.
5. **Restaurar desde backup limpio si hay alteración de datos confirmada.** Verifica la integridad de los backups antes de restaurar, y documenta qué se cambió para no perder evidencia útil.
6. **Forzar el restablecimiento de contraseñas de usuarios** si hay indicios de que se accedió a la tabla de credenciales, incluso si estaban hasheadas.
7. **Notificar según corresponda.** Dependiendo de la jurisdicción y el tipo de datos expuestos (datos personales, de pago), puede existir obligación legal de notificar a usuarios y/o autoridades de protección de datos dentro de un plazo determinado.
8. **Documentar la línea de tiempo del incidente** mientras los detalles están frescos: cuándo se detectó, qué se observó, qué acciones se tomaron y en qué orden.

---

## Prevención a largo plazo

- **Usar siempre consultas parametrizadas / prepared statements** (o un ORM que las implemente correctamente) — nunca concatenar input del usuario directamente en SQL.
- **Aplicar el principio de mínimo privilegio** a la cuenta de base de datos que usa la aplicación (sin permisos de `DROP`, `ALTER` o acceso a otras bases si no es necesario).
- **Validar y tipar estrictamente toda entrada** (rangos numéricos, longitudes, listas blancas de valores permitidos).
- **Desactivar mensajes de error detallados en producción**; loguearlos internamente, mostrar solo errores genéricos al usuario.
- **Implementar un WAF** con reglas actualizadas contra patrones de inyección conocidos.
- **Registrar y monitorear consultas anómalas** con alertas automáticas ante patrones de SQLi.
- **Realizar pruebas de penetración y análisis estático de código periódicamente**, especialmente antes de cada release mayor.
- **Mantener actualizado el motor de base de datos y los frameworks** usados, aplicando parches de seguridad sin demora.
- **Cifrar datos sensibles en reposo** (especialmente datos de pago, que idealmente no deberían almacenarse directamente sino delegarse a un proveedor PCI-DSS compliant).

---

## Checklist rápido

- [ ] Todas las consultas usan parámetros, no concatenación de strings
- [ ] La cuenta de BD de la app tiene privilegios mínimos
- [ ] Los mensajes de error en producción no revelan detalles internos
- [ ] Hay un WAF activo con reglas de SQLi habilitadas
- [ ] Existen alertas automáticas para patrones de tráfico anómalo
- [ ] Hay backups recientes y probados (restauración verificada)
- [ ] Existe un plan de respuesta a incidentes documentado y socializado con el equipo
- [ ] Las credenciales y secretos se rotan periódicamente

---

## ⚠️ Aviso

Este documento describe el proceso a **nivel conceptual** con fines de concientización y defensa (detección, respuesta y prevención). No incluye payloads, sintaxis de explotación ni pasos operativos de ataque. Para pruebas de seguridad reales, usa siempre entornos autorizados (pentesting con permiso explícito, bug bounty programs, o entornos de laboratorio como OWASP Juice Shop / DVWA).

---

*Última actualización: Junio 2026*
