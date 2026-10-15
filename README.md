# NORTE — Frontend E-commerce (ecommerce2)

Frontend Next.js 16.4 + React 19 + TypeScript que consume la API Laravel 12 existente en `../ecommerce-api`. El backend original conserva su código, base de datos y Stripe. Este directorio contiene el **frontend**; `../ecommerce-api` contiene el backend Laravel.

## Instalación y ejecución

Requisitos: Node.js 22.13+ (o una versión posterior compatible), pnpm, API Laravel funcionando y Stripe en modo de prueba. Se incluye `pnpm-lock.yaml` para reproducir las dependencias.

```bash
cd /Users/marloncastillo/ecommerce2
pnpm install
cp .env.example .env.local
# Editar .env.local con los valores de tu entorno.
pnpm dev
```

Abrir http://localhost:3060/catalogo. Si `.env.local` ya existe, editarlo sin sobrescribirlo. El entorno local preparado incluye la URL de la API y la clave **publicable** de Stripe del backend; ninguna clave secreta se copió.

En otra terminal, iniciar la API existente:

```bash
cd /Users/marloncastillo/ecommerce-api
php artisan serve --host=127.0.0.1 --port=8000
```

MySQL debe estar activo y la API previamente instalada según su propio README. No ejecutar migraciones destructivas ni volver a sembrar datos si la base ya está configurada.

Producción local:

```bash
pnpm build
pnpm start
```

El puerto del frontend es 3060 tanto en desarrollo como en producción local. El backend usa 8000. Para cambiarlo: `pnpm exec next dev -p 3070` y actualizar `APP_URL`.

## Variables de entorno

| Variable | Ejemplo / propósito | Exposición |
| --- | --- | --- |
| `API_BASE_URL` | `http://127.0.0.1:8000/api/v1` | Solo servidor; URL absoluta, sin slash final |
| `APP_URL` | `http://localhost:3060` | URL del frontend; usar HTTPS en despliegue |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Clave `pk_test_...` de la misma cuenta Stripe que usa Laravel | Pública, necesaria para Stripe.js |

`STRIPE_SECRET`, `STRIPE_WEBHOOK_SECRET`, credenciales MySQL y secreto JWT permanecen exclusivamente en el `.env` de Laravel. Nunca colocar secretos en variables `NEXT_PUBLIC_*`. No versionar `.env.local`. Cambios en la clave publicable requieren reiniciar desarrollo o recompilar producción.

## Uso

1. Explorar `/catalogo` y abrir `/catalogo/[id]`.
2. Agregar productos al carrito. Se conserva localmente en el navegador; no contiene tokens.
3. Registrarse en `/registro` o iniciar sesión en `/login`.
4. En `/checkout`, quitar productos o crear la orden. La API verifica stock y calcula precios definitivos; el cliente solo envía IDs y cantidades.
5. En `/checkout/[id]`, preparar el PaymentIntent y completar el Payment Element de Stripe.
6. Revisar `/confirmacion/[id]`. Solo muestra compra confirmada cuando Laravel devuelve `paid`.
7. Consultar `/pedidos` para ver el historial o retomar el pago de una orden pendiente.

La API no proporciona imágenes de producto: las tarjetas muestran una composición tipográfica local, no fotografías inventadas.

## Stripe y confirmación

Antes de probar pagos, usar claves **de prueba** en ambos proyectos y mantener activo el webhook:

```bash
stripe login
stripe listen --forward-to http://127.0.0.1:8000/api/v1/stripe/webhook
```

Configurar el secreto que muestra Stripe CLI en `STRIPE_WEBHOOK_SECRET` de Laravel y ejecutar `php artisan config:clear`. El secreto puede cambiar entre sesiones de CLI. La API escucha `payment_intent.succeeded`, `payment_intent.payment_failed` y `payment_intent.canceled`.

Para una prueba manual usar la tarjeta de prueba oficial `4242 4242 4242 4242`, una fecha futura y CVC de prueba. Nunca introducir datos de tarjetas reales durante la práctica. Stripe.js recibe los datos; este frontend y Laravel no reciben números de tarjeta.

El endpoint `POST /orders/{id}/payment-intent` crea o reutiliza el intento mediante la clave idempotente de la API. `confirmPayment()` realiza la confirmación y las autenticaciones adicionales. El retorno a `/confirmacion/[id]` **no prueba que el pago haya ocurrido**: se consulta la orden al backend. Si el webhook todavía no llegó, usar “Actualizar estado”. No se confía en parámetros `redirect_status` ni `payment_intent` de la URL.

**Validación pendiente:** no se ha ejecutado un cobro de prueba completo contra Stripe ni verificado la entrega real del webhook durante esta implementación. Se comprobó el manejo de fallos con una API simulada sin cargos ni cambios de inventario real.

## Arquitectura y archivos

| Archivo / directorio | Responsabilidad |
| --- | --- |
| `src/app/catalogo/page.tsx` | Server Component, listado público paginado con Suspense |
| `src/app/catalogo/[id]/page.tsx` | Detalle público, stock y 404 |
| `src/app/actions.ts` | Server Actions: registro, login, logout, orden, PaymentIntent y actualización de estado |
| `src/lib/api.ts` | Cliente REST exclusivo del servidor; Bearer desde cookie, timeout de 12 s, errores seguros |
| `src/lib/validation.ts` | Validación Zod de credenciales y carrito, formato de moneda y paginación |
| `src/lib/types.ts` | Contratos tipados según los Resources de Laravel |
| `src/proxy.ts` | Middleware de Next.js 16: redirección temprana sin cookie; no reemplaza la autorización de la API |
| `src/components/cart.tsx` | Estado local con useSyncExternalStore, hidratación y persistencia; vaciado solo tras crear la orden |
| `src/components/auth-form.tsx` | Formularios con useActionState, validación HTML y errores de servidor |
| `src/components/payment.tsx` | Payment Element y confirmación Stripe |
| `src/components/payment-loader.tsx` | Carga diferida de Stripe, limitada al checkout |
| `src/app/checkout/*` | Carrito y pago de una orden existente |
| `src/app/pedidos/page.tsx` | Historial privado paginado con Suspense |
| `src/app/confirmacion/[id]/page.tsx` | Estado de compra confirmado por la API |
| `loading.tsx` / `error.tsx` | Fallbacks de catálogo, checkout, historial y confirmación |
| `src/components/vitals.tsx` | Web Vitals en consola durante desarrollo |
| `src/app/globals.css` | Diseño responsive, foco visible y estilos sin fuentes externas |
| `tests/` | Validaciones y pruebas de integración reproducibles |
| `reports/` | Estado verificable de las pruebas y guía de Lighthouse |

## Seguridad, caché y resiliencia

- JWT en cookie `httpOnly`, `sameSite=lax`, `path=/`, duración según `expires_in`. `secure=true` cuando `APP_URL` usa HTTPS. La cookie se escribe en Server Actions, que Next.js permite para las mutaciones; el middleware `proxy.ts` aplica la protección inicial de rutas. Producción debe usar HTTPS y `APP_URL=https://...`.
- `requireUser()` valida con `/auth/me` y redirige al login si expira la sesión. Cada mutación lleva el JWT y Laravel vuelve a autenticar y autorizar al dueño de la orden.
- Protección CSRF de Server Actions mediante comparación Origin/Host, más SameSite. No se ampliaron los orígenes permitidos. Si hay un proxy inverso, configurar correctamente su host.
- React escapa el texto de la API. No se utiliza HTML arbitrario ni `dangerouslySetInnerHTML`.
- Todas las lecturas REST usan `cache: no-store`, apropiado para stock e información privada. No hay caché compartida de órdenes ni usuarios. `revalidatePath()` actualiza catálogo, historial, confirmación y navegación después de mutaciones.
- Listas paginadas y Server Components reducen JavaScript en el cliente. Suspense transmite primero la estructura; loading y error cubren esperas y fallos.
- Stripe se carga únicamente en la página de pago. No se descargan imágenes ni fuentes externas en el catálogo.
- Botones pendientes y bloqueo del formulario reducen envíos dobles. No se reintentan automáticamente los POST. Ante una respuesta incierta al crear una orden, revisar historial antes de reintentar.
- **Límite del backend existente:** la creación de órdenes no tiene idempotencia persistente. Un reenvío desde otra pestaña o una caída de red aún puede duplicar órdenes. Para producción debe añadirse una clave de idempotencia con restricción única en Laravel, junto con expiración de reservas y reposición de inventario. No se alteró la base original para introducir estas reglas.
- No se renueva el JWT silenciosamente: al expirar, se solicita login de nuevo. La API original conserva su rate limiting.

## Pruebas

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm build
```

Las pruebas unitarias validan cantidades, duplicados, registro y paginación. Para navegador se necesita Chrome local o `CHROME_PATH` apuntando a un ejecutable compatible.

```bash
# Con API real en 8000 y frontend de producción en 3060:
pnpm test:browser

# API simulada en 8099 + frontend temporal en 3061; sin tocar la base real:
pnpm test:flow
```

`test:flow` arranca y cierra sus propios servidores. Comprueba login, registro, cookie httpOnly, rutas privadas, creación de orden, errores de Stripe, confirmación independiente de la URL, logout y recuperación de errores. Los puertos 8099 y 3061 deben estar libres. `tests/mock-api.mjs` permite iniciar manualmente la misma API simulada, sin persistencia.

Estado exacto de ejecución: ver `reports/EVIDENCIA.md`. No confundir pruebas preparadas con pruebas ejecutadas.

## Lighthouse / Web Vitals

Con Laravel en 8000 y el frontend compilado ejecutándose en 3060:

```bash
pnpm build
pnpm start
# En otra terminal:
pnpm lighthouse
```

Se generan `reports/lighthouse-catalogo.report.html` y `.report.json`. Si Chrome no se detecta en macOS:

```bash
CHROME_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" pnpm lighthouse
```

Ejecutar tres veces sin extensiones ni otras auditorías simultáneas y reportar la mediana. Revisar LCP, CLS y TBT; Lighthouse de navegación no mide INP de usuarios reales. `useReportWebVitals` sirve para observar métricas en consola de desarrollo; un despliegue real necesitaría un receptor de telemetría y datos de campo.

**La evidencia Lighthouse todavía no está disponible:** Chrome fue bloqueado por permisos del entorno de ejecución (`bootstrap_check_in ... Permission denied`). No se inventaron puntajes. Ejecutar el comando anterior desde la terminal normal del equipo para generar los reportes pendientes.
