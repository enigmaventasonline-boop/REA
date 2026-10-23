# Especificación técnica · Módulo "Alertas mensuales" (Optopus)

> Documento para que Claude Code (u otro desarrollador) reconstruya el módulo dentro del stack de Optopus.
> `Optopus Alertas.dc.html` es solo la **maqueta de referencia**: tómala como guía visual y de lógica, no la copies tal cual.

---

## 1. Modelo de datos

### Regla (`alertas`)
```ts
type Severidad = 'alta' | 'media' | 'baja';
type Pagina = 'Todas' | 'Hoy' | 'Compras' | 'Ventas' | 'Empaque';

interface Alerta {
  id: string;
  on: boolean;            // activa / apagada
  name: string;           // título del banner
  msg: string;            // mensaje con variables {fecha} {dias} {mes} {mes_anterior}
  day: number | 'ult';    // 1–28 o 'ult' = último día del mes
  lead: number;           // días de anticipación (0–15)
  sev: Severidad;
  pages: Pagina[];        // 'Todas' excluye a las demás
  link?: string;          // texto del enlace (opcional)
  href?: string;          // ruta interna (agregar en producción)
}
```

### Estado por período (`alertas_periodo`)
```ts
interface AlertaPeriodo {
  alertaId: string;
  periodo: string;        // 'YYYY-MM'
  hecha?: { por: string; en: string };   // usuario + timestamp ISO
  pospuestaHasta?: string;               // 'YYYY-MM-DD'
}
```
La clave natural es `alertaId + periodo`, así cada alerta vuelve sola el mes siguiente.

### Datos iniciales
| id | name | day | lead | sev | pages |
|---|---|---|---|---|---|
| f29 | Declarar y pagar el F29 | 20 | 5 | alta | Todas |
| dte46 | Emitir facturas de compra del mes | ult | 3 | media | Compras, Hoy |
| previred | Pagar cotizaciones (Previred) | 13 | 2 | alta | Todas |
| ml | Revisar factura de Mercado Libre | 5 | 0 | baja | Ventas, Hoy |
| stock | Conteo de stock en bodega | 28 | 2 | baja (on:false) | Hoy |

---

## 2. Lógica (función pura, servidor o cliente)

```js
const lastDay = (y, m) => new Date(y, m + 1, 0).getDate();

function vencimiento(alerta, hoy) {
  const y = hoy.getFullYear(), m = hoy.getMonth();
  const d = alerta.day === 'ult' ? lastDay(y, m) : Math.min(alerta.day, lastDay(y, m));
  return new Date(y, m, d);
}

function estado(alerta, hoy, periodoState) {
  const due = vencimiento(alerta, hoy);
  const left = Math.round((due - hoy) / 86400000);           // días que faltan (negativo = vencida)
  if (!alerta.on) return { kind: 'apagada', show: false, due, left };
  if (periodoState?.hecha) return { kind: 'hecha', show: false, due, left };
  const pospuesta = periodoState?.pospuestaHasta && periodoState.pospuestaHasta > isoDate(hoy);
  let kind = left < 0 ? 'vencida' : left <= alerta.lead ? 'aviso' : 'espera';
  let show = kind === 'vencida' || kind === 'aviso';
  if (show && pospuesta) { show = false; kind = 'pospuesta'; }
  return { kind, show, due, left };
}
```

### Orden y límite de banners
1. Solo `show === true` y página actual ∈ `pages` (o `pages` incluye `'Todas'`).
2. Ordenar por `left` ascendente; en caso de empate: alta → media → baja.
3. Mostrar como máximo `maxBanners` (por defecto 3).

### Reemplazo de variables
| Variable | Valor |
|---|---|
| `{fecha}` | `"20 de octubre"` (día + mes en español) |
| `{dias}` | `max(0, left)` |
| `{mes}` | mes actual en minúsculas |
| `{mes_anterior}` | mes anterior en minúsculas |

### Texto del indicador
- `left < 0` → `Venció hace N día(s)`
- `left === 0` → `Vence hoy`
- `left === 1` → `Vence mañana`
- otro → `Vence en N días`

### Estado en la tabla de configuración
`Apagada` · `Hecha` · `Vencida` · `Mostrando banner` · `Pospuesta` · `Avisa el {vencimiento − lead}`

---

## 3. API

```
GET    /api/alertas                         → Alerta[]
POST   /api/alertas                         ← Alerta (sin id)      → 201 Alerta
PATCH  /api/alertas/:id                     ← Partial<Alerta>      → Alerta
DELETE /api/alertas/:id                     → 204
POST   /api/alertas/:id/hecha               ← { periodo }          → 200   (usuario = sesión)
POST   /api/alertas/:id/posponer            ← { periodo, hasta }   → 200
GET    /api/alertas/activas?pagina=Hoy      → Banner[]             (ya filtrado, ordenado y limitado)
```
```ts
interface Banner { id; title; text; sev; when; link?; href?; periodo }
```
La fecha "hoy" se calcula en el **servidor**, en zona `America/Santiago`.

---

## 4. Componentes de UI

| Componente | Dónde | Responsabilidad |
|---|---|---|
| `<AlertBanners pagina="...">` | Layout global, arriba del contenido de cada página | Llama a `/activas`, muestra los banners y ejecuta *hecha* y *posponer* |
| `<AlertBell>` | Barra superior, a la izquierda del usuario | Contador de alertas activas (punto rojo con número) |
| `AlertasPage` | Configuración › Alertas | Tabla de reglas + panel de edición |
| `AlertRow` | Tabla | Interruptor, punto de severidad, título, páginas, día, aviso, estado |
| `AlertEditor` | Panel derecho | Título, mensaje, día (select 1–28 + último), aviso (− / +), severidad (segmentado), páginas (chips), resumen del próximo vencimiento |

En la maqueta, el selector **"Ver como el día"** sirve solo para probar. No va en producción, o va solo para el rol administrador.

---

## 5. Tokens visuales (estilo admin Shopify)

| Uso | Valor |
|---|---|
| Fondo de la app | `#F1F1F1` |
| Barra superior | `#1A1A1A`, alto 56px |
| Menú lateral | `#EBEBEB`, ancho 200px, ítem activo `#FFFFFF` |
| Tarjeta | `#FFFFFF`, radio 12px, sombra `0 1px 0 rgba(26,26,26,.07), 0 0 0 1px rgba(26,26,26,.06)` |
| Texto principal / secundario | `#303030` / `#616161` |
| Botón primario | `#303030`, texto blanco, radio 8px, alto 28–30px |
| Enlace | `#005BD3` |
| Tipografía | system-ui (`-apple-system, Segoe UI, Roboto`), base 13px |

### Severidad
| sev | Punto | Fondo del banner | Borde | Texto | Pastilla |
|---|---|---|---|---|---|
| alta | `#E51C00` | `#FEE9E8` | `#FDB7B0` | `#5F1405` | `#FDD2CC` |
| media | `#C05717` | `#FFF1C2` | `#F5D985` | `#4F3A00` | `#FFE7A0` |
| baja | `#0094D5` | `#EAF4FF` | `#B9DAF7` | `#00365A` | `#D3E8FC` |

### Banner (estructura)
```
[●!]  Título  [Vence en 3 días]
      Mensaje con variables reemplazadas
      [Marcar como hecho]  [Recordar mañana]  Enlace →
```
Radio 12px, padding 12×14px, borde interno de 1px con el color de borde.

---

## 6. Tarea para Claude Code

1. Crear las tablas `alertas` y `alertas_periodo` con los datos iniciales de la sección 1.
2. Implementar la lógica de la sección 2 en un servicio con pruebas unitarias. Casos mínimos:
   - día 31 en febrero
   - `'ult'`
   - `lead = 0`
   - vencida
   - hecha
   - pospuesta
   - cambio de mes
3. Exponer la API de la sección 3, protegida con la sesión de Optopus.
4. Agregar `<AlertBanners>` al layout global y `<AlertBell>` a la barra superior.
5. Construir la página Configuración › Alertas siguiendo la maqueta y los tokens de la sección 5.
6. (Opcional) Un job diario a las 08:00 (Santiago) que avise por correo o WhatsApp las alertas que empiezan a mostrarse ese día.
