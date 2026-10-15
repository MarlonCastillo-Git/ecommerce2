# Evidencia de verificación — 15 de octubre de 2026

## Ejecutado correctamente

- `pnpm lint`: sin errores.
- `pnpm typecheck`: sin errores.
- `pnpm test`: 3 pruebas unitarias aprobadas.
- `pnpm build`: compilación de producción Next.js 16.4.1 aprobada.
- API Laravel real: GET `/api/v1/products` devolvió HTTP 200 y 8 productos.
- Navegador integrado: catálogo real, detalle, agregar al carrito y redirección del checkout al login comprobados.
- Navegador integrado contra API simulada: login, creación de orden, carrito vaciado, error de preparación del pago y confirmación pendiente aunque la URL contenga `redirect_status=succeeded` comprobados. También se verificaron historial de órdenes pendientes y cierre de sesión.

## Pendiente / bloqueado

- Lighthouse: la ejecución se intentó contra producción local. Chrome y Chromium Headless Shell no pudieron iniciar por la restricción de macOS `bootstrap_check_in ... Permission denied (1100)`. No existe informe con métricas válidas.
- Scripts Playwright: preparados, pero su ejecución automática quedó bloqueada por el mismo permiso del navegador. Parte del flujo se verificó con el navegador integrado, como se detalla arriba.
- Pago Stripe completo y entrega real del webhook: no ejecutados. No se generaron cobros ni órdenes en la base real.

## Reproducir Lighthouse

Con la API y el frontend activos:

```bash
pnpm lighthouse
```

El comando guarda HTML y JSON en este directorio. Ejecutarlo desde una terminal del usuario que pueda iniciar Chrome. El README explica la configuración completa.
