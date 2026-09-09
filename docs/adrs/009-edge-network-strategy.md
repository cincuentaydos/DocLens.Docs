# ADR-009 — Estrategia de Red y Edge

**Estado:** Aceptada (2026-09-07)

---

## Decisión

Usar una capa de entrada **100% nativa de AWS**: Amazon Route 53 (DNS) + Amazon CloudFront (CDN/entrada de la aplicación) + AWS WAF (reglas de protección y rate limiting) + AWS Shield Standard (protección DDoS) + AWS Certificate Manager (certificados TLS).

Esto reemplaza un boceto inicial de arquitectura que consideraba **Cloudflare** como capa de borde delante de AWS.

---

## Contexto

El primer boceto de arquitectura de DocLens contemplaba Cloudflare como capa de entrada (CDN, WAF, DNS) delante de la infraestructura AWS. Al formalizar la arquitectura V1, se evaluó si mantener esa capa externa o consolidar todo dentro de AWS.

## Opciones consideradas

### Opción 1 — Cloudflare delante de AWS

**Fortalezas:** WAF y CDN de Cloudflare son maduros; separación entre proveedor de borde y proveedor de cómputo.

**Debilidades:** un proveedor adicional que gestionar (cuenta, facturación, configuración, observabilidad separada). La autenticación (Cognito) queda expuesta a internet de todos modos, por lo que Cloudflare no aporta una capa de auth adicional. Duplica funciones que CloudFront + WAF + Shield ya cubren nativamente dentro de AWS.

### Opción 2 — Capa de borde nativa de AWS (elegida)

**Fortalezas:**

- Un solo proveedor — menor complejidad operativa, una sola consola, un solo modelo de IAM.
- CloudFront + WAF + Shield Standard + ACM cubren CDN, reglas de protección/rate limiting, DDoS y TLS sin salir de AWS.
- Integración directa con Terraform (mismo proveedor `aws` para toda la infraestructura, sin un segundo proveedor Terraform para Cloudflare).
- Route 53 mantiene la resolución DNS dentro de AWS, simplificando el enrutamiento hacia CloudFront.

**Debilidades:**

- Shield Standard (incluido, gratuito) ofrece menos capacidades que Shield Advanced o las protecciones DDoS de nivel empresarial de Cloudflare — aceptable para el volumen esperado en V1.
- Menor flexibilidad de reglas WAF que algunas ofertas de terceros especializados.

---

## Comparación

| Dimensión | Cloudflare + AWS | Solo AWS (elegida) |
|---|---|---|
| Proveedores a gestionar | 2 | 1 |
| Complejidad de Terraform | 2 providers | 1 provider |
| DDoS | Cloudflare (nivel alto) | Shield Standard (básico, incluido) |
| CDN | Cloudflare | CloudFront |
| WAF | Cloudflare | AWS WAF |
| TLS | Cloudflare o ACM | ACM |
| Costo adicional | Sí (plan Cloudflare) | No (incluido en servicios AWS ya usados) |

---

## Enfoque elegido

```
Cliente → Route 53 (DNS) → CloudFront (CDN + entrada de la app)
                                ├── WAF (reglas + rate limiting)
                                ├── Shield Standard (DDoS)
                                └── ACM (certificados TLS)
```

Amazon Cognito se expone directamente a internet para la autenticación (login/password, emisión de JWT), fuera de la ruta de CloudFront — ver `architecture.md`.

## Consecuencias

- **Terraform:** un solo proveedor (`aws`) gestiona Route 53, CloudFront, WAF y ACM — sin necesidad del proveedor Terraform de Cloudflare.
- **Observabilidad:** logs y métricas de la capa de borde quedan centralizados en CloudWatch, junto con el resto del sistema.
- **Revisar si:** el volumen de tráfico o los requisitos de protección DDoS de nivel empresarial justifican evaluar Shield Advanced en el futuro.

## Preguntas abiertas

- ¿Se necesita Shield Advanced antes de un lanzamiento público más amplio?
- ¿Qué reglas gestionadas de WAF (AWS Managed Rules) se activan primero — reglas OWASP genéricas, bot control, o ambas?
