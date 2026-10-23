---
name: ingenieria-inversa-modulo
description: Analiza cómo funciona una app, módulo, pantalla o link (SaaS, competidor, captura, repo, documentación) y genera una especificación técnica en español (README.es.md), lista para que Claude Code construya un módulo PROPIO equivalente dentro del proyecto del usuario (Optopus u otro). Úsala cuando el usuario diga "ingeniería inversa", "analiza cómo funciona X y adáptalo", "quiero esto como módulo", "hazme el contexto/README para Claude Code", o cuando entregue un link o captura de otra herramienta y pida algo similar.
---

# Ingeniería inversa → especificación de módulo propio

## Objetivo
Entender **qué hace** y **cómo se comporta** una herramienta de referencia y convertirlo en una especificación técnica original para el proyecto del usuario. El resultado final es siempre un `README.es.md` que Claude Code pueda ejecutar paso a paso.

## Límites (no negociables)
- Se analiza el **comportamiento observable**: flujo, pantallas, reglas de negocio y datos visibles, a partir de la documentación pública, capturas, demos o del uso normal.
- **No** se descompila, no se extrae código privado, no se evaden logins o pagos, y no se copian textos, logos ni marca.
- Si el link pide iniciar sesión, se trabaja con la documentación pública, la ayuda oficial, los videos y las capturas que entregue el usuario, y se avisa qué no se pudo ver.
- El diseño final usa la identidad del **proyecto del usuario**, no la de la referencia.

---

## Paso 1 · Reunir fuentes
Según lo que entregue el usuario:

| Entrada | Qué hacer |
|---|---|
| Link público | `web_fetch` de la página + `web_search` de "<producto> cómo funciona", "<producto> help center", "<producto> API" |
| Link con login | Buscar documentación pública y pedir capturas de las pantallas clave |
| Captura / video | Leer cada pantalla: campos, estados, botones y mensajes |
| Repo GitHub | Leer README, modelos, rutas y esquemas (código abierto: respetar la licencia y citarla) |
| Solo el nombre | `web_search` de la documentación oficial, precios, integraciones y reseñas |

Anota en una lista corta: **qué problema resuelve**, **para quién** y **qué integra**.

## Paso 2 · Descomponer
Extrae siempre estas 7 capas. Si una no aplica, dilo explícitamente.

1. **Flujo principal:** los pasos del usuario, del 1 al N.
2. **Pantallas:** lista, con sus componentes y estados (vacío, cargando, error, éxito).
3. **Entidades de datos:** objetos, campos y relaciones.
4. **Reglas de negocio:** cálculos, validaciones, umbrales, fechas e impuestos.
5. **Integraciones:** APIs externas, archivos, correos, webhooks, cron.
6. **Permisos:** roles y qué puede hacer cada uno.
7. **Casos borde:** fallos, duplicados, límites y cambios de mes, moneda o zona horaria.

## Paso 3 · Adaptar al proyecto del usuario
- Revisa qué ya existe en el proyecto: integraciones, módulos, design system, `github.md` y `CLAUDE.md`. **Reutiliza los datos que el proyecto ya tiene** antes de pedir integraciones nuevas.
- Elimina lo que no aporta al usuario y agrega lo que su negocio necesita (país, moneda, canales, normativa local).
- Si la regla de negocio depende de normativa (impuestos, SII, laboral), márcala con "Validar con contador/abogado".

## Paso 4 · Maqueta (opcional, recomendado)
Construye la pantalla principal como Design Component (`<Proyecto> <Módulo>.dc.html`) con el sistema visual del proyecto, datos de ejemplo realistas y la lógica funcionando. Sirve como referencia visual para el README.

## Paso 5 · Generar `README.es.md`
Guárdalo en `<proyecto>-<modulo>/README.es.md` junto con la maqueta y entrega el zip con `present_fs_item_for_download`.

Usa **exactamente** esta plantilla:

```markdown
# Especificación técnica · Módulo "<Nombre>" (<Proyecto>)

> Referencia analizada: <producto/link> — solo comportamiento observable.
> Maqueta: `<archivo>.dc.html` (guía visual y de lógica, no copiar tal cual).

## 0. Resumen
- Problema que resuelve:
- Usuario:
- Resultado esperado:

## 1. Flujo
1. …
2. …

## 2. Modelo de datos
```ts
interface Entidad { … }
```
Datos iniciales / ejemplo: tabla.

## 3. Reglas de negocio
Fórmulas, validaciones y estados en una función pura (pseudocódigo JS). Tabla de estados → texto visible.

## 4. Integraciones
| Fuente | Método | Frecuencia | Qué entrega |

## 5. API
```
METHOD /ruta  ← body  → respuesta
```

## 6. Componentes de UI
| Componente | Dónde | Responsabilidad |

## 7. Tokens visuales
Colores, radios, tipografía y estados, tomados del design system del proyecto.

## 8. Casos borde y pruebas
Lista de casos que deben tener test unitario.

## 9. Pendientes / a validar
Normativa, credenciales, decisiones del negocio.

## 10. Tarea para Claude Code
1. Crear modelos y datos iniciales.
2. Implementar la lógica con tests.
3. Exponer la API con la sesión del proyecto.
4. Construir la UI según la maqueta y los tokens.
5. (Opcional) Jobs, notificaciones y siguiente fase.
```

## Paso 6 · Cierre en el chat
Máximo 5 líneas:
- qué se analizó y qué no se pudo ver
- qué se adaptó al proyecto
- qué falta validar (normativa o credenciales)

---

## Ejemplos ya aplicados en este proyecto
- **Empaque con evidencia:** flujo tipo verificador de empaque → `optopus-empaque/`
- **Compras extranjero / DTE 46:** flujo tipo recuperador de IVA de servicios digitales → `optopus-compras-extranjero/`
- **Alertas mensuales:** recordatorios recurrentes con banner → `optopus-alertas/README.es.md` (usar como modelo de formato)
