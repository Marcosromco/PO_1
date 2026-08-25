# FASE 1 — Producto, Arquitectura y Plan Técnico

Marketplace de servicios para el hogar en México (plomería, electricidad, carpintería, jardinería, extensible a más categorías). Lanzamiento inicial: CDMX y zona metropolitana.

Este documento es el entregable de Fase 1. No contiene código de aplicación — es la base sobre la que se construirán las Fases 2 a 12.

---

## 1. Resumen del producto

Un cliente con un problema en casa debe poder resolverlo sin salir de la app:

```
solicitud → matching → profesional → presupuesto → aprobación → trabajo → pago → garantía → review
```

Tres roles: **Cliente**, **Profesional**, **Administrador**. Tres modos de contratación: **Urgente**, **Programado**, **Cotización**. El producto ataca 7 fricciones de confianza descritas por el usuario (quién entra a mi casa, disponibilidad, precio, calidad, seguridad, garantía, comodidad) — cada decisión de UX y de arquitectura de este documento existe para resolver una de ellas, no como feature aislada.

---

## 2. Decisiones críticas (y por qué)

| Decisión | Elección | Razón |
|---|---|---|
| App móvil única vs. dos apps (cliente/profesional) | **Una sola app Expo**, con navegación completamente distinta según `role` tras login | Con un equipo chico, mantener 2 apps nativas duplica CI/CD, releases y QA. La lógica de negocio (auth, notificaciones, chat) se comparte. Uber/Rappi separaron sus apps *después* de escalar, no antes. Se puede separar más adelante sin tocar el backend. |
| Backend monolito modular vs. microservicios | **Monolito modular (NestJS)**, con módulos con fronteras claras (matching, pagos, disputas...) | Microservicios agregan complejidad operativa (colas, service discovery, tracing distribuido) que no se justifica al tamaño de un MVP. Los módulos de Nest ya fuerzan separación de responsabilidades; extraer un módulo a servicio propio después es viable porque no hay acoplamiento oculto. |
| Geolocalización | **PostgreSQL + extensión PostGIS** desde el día uno | El matching por distancia es núcleo del producto, no un extra. Agregar PostGIS después implica migrar datos geográficos en producción. Activarlo ahora no añade complejidad de infraestructura (mismo Postgres). |
| Dinero | Enteros en **centavos (MXN)**, nunca `float` | Estándar de la industria para evitar errores de redondeo en pagos/comisiones. |
| Pagos | **Interfaz `PaymentGateway` desacoplada** + implementación mock funcional para desarrollo/tests + adaptador real conectable | No se puede depender de credenciales de un proveedor de pagos mexicano sin que el usuario las provea. La arquitectura no debe bloquearse por eso. |
| Matching | **Servicio de scoring modular y configurable**, no "el más cercano" | Pedido explícito del usuario; además evita razas de carrera y mejora calidad de asignación. |
| Borrado de datos | **Soft delete** (`deletedAt`) en entidades con implicaciones legales/financieras (User, Service, Payment) | Requisito de auditoría y disputas: nunca se debe perder el rastro de un servicio pagado. |
| Documentos sensibles | **Nunca en la base de datos ni en URLs públicas** — se guardan en storage privado y se sirven con URLs firmadas de corta duración | Requisito de seguridad explícito (INE, selfie, cuentas bancarias). |

---

## 3. Stack tecnológico

| Capa | Tecnología | Por qué |
|---|---|---|
| **Mobile** (cliente + profesional) | **React Native + Expo** (TypeScript) | Un solo lenguaje (TS) en todo el stack → tipos y validaciones (Zod) compartidos con el backend y la web admin. Expo da OTA updates, EAS Build gestionado, y módulos maduros para lo que necesitamos (`expo-location`, `expo-camera`, `expo-notifications`, `expo-image-picker`). Mayor pool de talento TS/JS en México que Dart/Flutter, lo que importa para poder escalar el equipo. Flutter es igualmente válido técnicamente, pero fragmenta el stack (Dart aparte) sin beneficio claro para este producto. |
| **Backend** | **NestJS + TypeScript + PostgreSQL** | Nest da estructura (módulos, guards, interceptors, pipes) que mapea 1:1 con lo que pide el producto: RBAC, validación, máquina de estados, WebSockets. Curva de adopción baja al compartir TS con mobile/web. |
| **ORM** | **Prisma** | Migraciones versionadas, tipado end-to-end, buen soporte de PostGIS vía `Unsupported` + raw queries donde haga falta. |
| **Web admin** | **Next.js + TypeScript + Tailwind + shadcn/ui** | SSR para tablas grandes (disputas, servicios), DX rápida, comparte `packages/shared-types` con el resto del monorepo. |
| **Realtime** | **WebSockets (Socket.IO vía `@nestjs/websockets`)**, con adaptador Redis para escalar horizontalmente | Chat, tracking de ubicación del profesional y push de cambios de estado necesitan push server→cliente; Socket.IO es la opción más simple y probada para esto en el ecosistema Node. |
| **Colas / jobs** | **BullMQ + Redis** | Timeouts de matching (si nadie acepta en N segundos, reintenta con más profesionales), auto-confirmación de servicio tras N horas, envío de notificaciones, recordatorios. |
| **Cache / pub-sub** | **Redis** | Doble uso: cola (BullMQ) y adaptador de Socket.IO. |
| **Storage** | **S3-compatible (AWS S3)**, dos buckets/prefijos: `public` (fotos de perfil, fotos de trabajos) y `private` (INE, selfie, comprobantes) servido con URLs firmadas | Requisito explícito de no exponer documentos privados. |
| **Mapas / geocoding / ETA** | **Google Maps Platform** (Maps SDK para mobile, Directions API para ETA, Geocoding API) | Estándar en México, buena cobertura de direcciones y tráfico en tiempo real. |
| **Pagos** | Interfaz `PaymentGateway` propia + **mock** funcional ahora; adaptador recomendado para producción: **Stripe** (tarjeta + Stripe Connect para payouts a profesionales) con opción de agregar **Conekta/OpenPay/Mercado Pago** para OXXO/SPEI más adelante sin tocar el dominio | Ver sección 12. |
| **Notificaciones push** | **Expo Notifications** (envuelve FCM/APNs) | Integración directa con Expo, sin necesidad de configurar Firebase por separado en MVP. |
| **Email** | Interfaz `EmailProvider` + adaptador **Resend** o **SendGrid** (a definir con credenciales del usuario) | |
| **SMS / OTP / WhatsApp** | Interfaz `SmsProvider`, mock en dev, adaptador **Twilio** recomendado para prod (Fase 2+) | |
| **Monorepo** | **pnpm workspaces + Turborepo** | Build cacheado, tareas compartidas, un solo `pnpm install`. |
| **Infraestructura** | Local: **Docker Compose** (Postgres+PostGIS, Redis, API, admin-web). Prod: contenedores en **Railway o Render** para MVP (rápido, barato, sin ops); migración a AWS (ECS/Fargate + RDS + S3) cuando el volumen lo justifique | Evita sobreingeniería de infraestructura en el MVP; el contenedor Docker es el mismo, solo cambia dónde corre. |
| **Auth** | JWT (access 15 min + refresh 30 días, rotación), guards RBAC en Nest, OTP por SMS para verificación de teléfono | |
| **Testing** | **Jest** (unit + integration backend), **Supertest** (API), **Playwright** (E2E web admin + flujos críticos) | |
| **Observabilidad** | Logs estructurados (Pino), audit log en base de datos para acciones sensibles, Sentry para errores (opcional, requiere API key del usuario) | |

---

## 4. Arquitectura de alto nivel

```
                         ┌─────────────────────────┐
                         │   Mobile App (Expo)      │
                         │  Cliente ↔ Profesional    │
                         └────────────┬─────────────┘
                                      │ HTTPS / WSS
                         ┌────────────┴─────────────┐
                         │     Admin Web (Next.js)   │
                         └────────────┬─────────────┘
                                      │
                    ┌─────────────────┴──────────────────┐
                    │            API Gateway              │
                    │        NestJS (REST + WS)           │
                    │  Auth · RBAC · Rate limiting · Val.  │
                    └───┬───────┬───────┬───────┬────────┘
                        │       │       │       │
          ┌─────────────┘   ┌───┘   ┌───┘       └──────────┐
          ▼                 ▼       ▼                       ▼
  ┌───────────────┐ ┌─────────────┐ ┌────────────┐  ┌───────────────┐
  │ Módulos de     │ │  Matching   │ │  Payments  │  │  Notifications │
  │ dominio (Nest) │ │  Engine     │ │  Gateway   │  │  (push/email)  │
  │ users/services/│ │ (scoring    │ │ (interfaz +│  │                │
  │ quotes/reviews │ │  modular)   │ │  mock/prod)│  │                │
  └───────┬────────┘ └──────┬──────┘ └─────┬──────┘  └───────┬────────┘
          │                 │              │                  │
          ▼                 ▼              ▼                  ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                    PostgreSQL + PostGIS (Prisma)                   │
  └───────────────────────────────────────────────────────────────────┘
          │                                              │
          ▼                                              ▼
  ┌───────────────┐                            ┌───────────────────┐
  │  Redis         │                            │  S3 (public/       │
  │ (BullMQ + WS   │                            │  private buckets)  │
  │  pub/sub)      │                            └───────────────────┘
  └───────────────┘
```

Todo tráfico externo (mapas, pagos, SMS, push) pasa por adaptadores en `packages`/módulos de infraestructura, nunca se llama directamente desde la lógica de dominio — así el dominio no sabe si el pago es mock o real.

---

## 5. Estructura de carpetas (monorepo)

```
po-1/
├── apps/
│   ├── api/                        # NestJS backend
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/
│   │   │   │   ├── users/
│   │   │   │   ├── customers/
│   │   │   │   ├── providers/
│   │   │   │   │   ├── onboarding/
│   │   │   │   │   └── verification/
│   │   │   │   ├── addresses/
│   │   │   │   ├── categories/
│   │   │   │   ├── service-requests/
│   │   │   │   ├── services/        # máquina de estados
│   │   │   │   ├── matching/        # scoring engine
│   │   │   │   ├── quotes/
│   │   │   │   ├── payments/
│   │   │   │   │   ├── gateway/     # interfaz + adapters (mock, stripe)
│   │   │   │   │   └── payouts/
│   │   │   │   ├── commissions/
│   │   │   │   ├── reviews/
│   │   │   │   ├── chat/            # gateway WS
│   │   │   │   ├── media/           # storage (S3 signed URLs)
│   │   │   │   ├── notifications/
│   │   │   │   │   └── channels/    # push, email, sms adapters
│   │   │   │   ├── disputes/
│   │   │   │   ├── warranty/
│   │   │   │   ├── safety/          # reportes, bloqueos
│   │   │   │   ├── admin/
│   │   │   │   ├── audit-log/
│   │   │   │   └── common/          # guards, interceptors, filters, decorators
│   │   │   ├── main.ts
│   │   │   └── app.module.ts
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── test/
│   │
│   ├── mobile/                     # Expo app (cliente + profesional)
│   │   ├── app/                    # expo-router
│   │   │   ├── (customer)/
│   │   │   ├── (provider)/
│   │   │   └── (auth)/
│   │   ├── components/
│   │   ├── features/                # lógica por dominio (mismo split que backend)
│   │   ├── hooks/
│   │   ├── services/api/            # cliente HTTP + WS tipado
│   │   └── store/                   # estado global (Zustand)
│   │
│   └── admin-web/                  # Next.js
│       ├── app/
│       │   ├── (dashboard)/
│       │   │   ├── users/
│       │   │   ├── providers/
│       │   │   ├── services/
│       │   │   ├── disputes/
│       │   │   ├── categories/
│       │   │   ├── commissions/
│       │   │   ├── warranty/
│       │   │   └── metrics/
│       │   └── login/
│       └── components/
│
├── packages/
│   ├── shared-types/                # DTOs y schemas Zod compartidos
│   ├── config/                      # eslint/tsconfig base
│   └── ui/                          # tokens de diseño compartidos (opcional, fase 2)
│
├── infra/
│   ├── docker-compose.yml
│   └── docker/
│       ├── api.Dockerfile
│       └── admin-web.Dockerfile
│
├── docs/
│   └── FASE-1-ARQUITECTURA.md      # este documento
│
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

---

## 6. Esquema de base de datos

Motor: **PostgreSQL 15+ con PostGIS**. ORM: Prisma (el `schema.prisma` completo se construye en Fase 2; aquí el modelo conceptual). Todas las entidades tienen `id` (UUID), `createdAt`, `updatedAt`; las que lo requieren tienen `deletedAt` (soft delete).

| Entidad | Campos clave | Relaciones |
|---|---|---|
| **User** | email? (unique), phone (unique), passwordHash?, role (`CUSTOMER`\|`PROVIDER`\|`ADMIN`), adminRole? (`SUPERADMIN`\|`SUPPORT`\|`FINANCE`\|`TRUST_SAFETY`), status (`ACTIVE`\|`SUSPENDED`\|`BANNED`\|`PENDING_VERIFICATION`), firstName, lastName, avatarUrl, phoneVerifiedAt, emailVerifiedAt | 1–1 CustomerProfile, 1–1 ProviderProfile |
| **CustomerProfile** | userId (fk, unique) | 1–N Address, 1–N ServiceRequest, 1–N FavoriteProvider |
| **Address** | ownerId (fk User), label, street, extNumber, intNumber, neighborhood, city, state, zip, **geo** (`Point`, PostGIS), instructions, isDefault | usada por ServiceRequest/Service (snapshot) |
| **ProviderProfile** | userId (fk, unique), legalName, birthDate, bio, yearsExperienceTotal, verificationStatus (`PENDING`\|`IN_REVIEW`\|`APPROVED`\|`REJECTED`\|`SUSPENDED`), ratingAvg, ratingCount, completedServicesCount, cancellationRate, punctualityScore, acceptanceRate, isOnline, currentGeo (`Point`), currentGeoUpdatedAt, payoutAccountToken (tokenizado, nunca CLABE en claro), joinedAt | 1–N ProviderVerification, 1–N ProviderZone, 1–N ProviderSkill |
| **ProviderVerification** | providerProfileId (fk), type (`ID_DOCUMENT`\|`SELFIE`\|`ADDRESS_PROOF`\|`BANK_ACCOUNT`\|`BACKGROUND_CHECK`\|...— extensible), documentKey (ruta privada en S3), status (`SUBMITTED`\|`APPROVED`\|`REJECTED`), reviewedBy (fk User admin), reviewedAt, rejectionReason | tabla extensible: agregar un nuevo `type` no requiere migración |
| **ProviderZone** | providerProfileId (fk), city, state, centerGeo (`Point`), radiusKm | define dónde recibe trabajos |
| **Category** | name, slug (unique), icon, description, isActive, sortOrder | 1–N Subcategory |
| **Subcategory** | categoryId (fk), name, slug, isActive, sortOrder, requiresQuoteApproval (bool), warrantyDays (default, override por categoría) | |
| **ProviderSkill** | providerProfileId (fk), categoryId (fk), subcategoryId? (fk), yearsExperience, isPrimary | join profesional↔categoría |
| **ServiceRequest** | customerId (fk), categoryId (fk), subcategoryId? (fk), type (`URGENT`\|`SCHEDULED`\|`QUOTE_REQUEST`), description, addressId (fk), scheduledWindowStart?, scheduledWindowEnd?, status (`OPEN`\|`MATCHING`\|`ASSIGNED`\|`CONVERTED`\|`EXPIRED`\|`CANCELLED`) | 1–N RequestMedia, 1–N Quote (cotizaciones competidas), 1–1 Service (al convertirse) |
| **RequestMedia** | serviceRequestId (fk), url, type (`PHOTO`\|`VIDEO`) | |
| **Service** | serviceRequestId (fk), customerId (fk), providerId? (fk), categoryId, subcategoryId?, type, status (ver §7 máquina de estados), addressId (fk, snapshot), scheduledAt?, pinHash, pinGeneratedAt, pinVerifiedAt, startedAt, completedAt, confirmedAt, cancelledAt, cancelledBy?, cancellationReason?, warrantyExpiresAt? | 1–N ServiceStatusHistory, 1–N Quote, 1 Chat, 1–N ServicePhoto, 1 Payment, 0–N Review, 0–N Dispute, 0–N WarrantyClaim |
| **ServiceStatusHistory** | serviceId (fk), fromStatus, toStatus, changedBy (fk User), reason? | auditoría de la máquina de estados |
| **Quote** | serviceId? (fk) / serviceRequestId? (fk, cotización competida), providerId (fk), type (`ESTIMATE`\|`FINAL`), visitFee, laborAmount, materialsAmount, totalAmount (centavos), currency (`MXN`), status (`PENDING`\|`APPROVED`\|`REJECTED`\|`EXPIRED`\|`SUPERSEDED`), notes, approvedBy?, approvedAt? | 1–N QuoteItem |
| **QuoteItem** | quoteId (fk), description, quantity, unitPrice, subtotal, type (`LABOR`\|`MATERIAL`\|`FEE`) | |
| **CommissionRule** | scope (`GLOBAL`\|`CATEGORY`\|`PROVIDER`\|`PROMOTION`), categoryId?, providerId?, promotionCode?, percentage, fixedFee?, validFrom?, validTo?, isActive, priority | resuelto en orden de especificidad al calcular Payment |
| **Payment** | serviceId (fk, unique), customerId (fk), grossAmount, materialsAmount, commissionAmount, taxAmount, netProviderAmount, currency, provider (`MOCK`\|`STRIPE`\|...), providerPaymentIntentId, method, status (`PENDING`\|`AUTHORIZED`\|`CAPTURED`\|`FAILED`\|`REFUNDED`\|`PARTIALLY_REFUNDED`), capturedAt | 1–N Refund, 1 Payout (o N si se agrupan) |
| **Payout** | providerId (fk), paymentId (fk), amount, status (`PENDING`\|`PROCESSING`\|`PAID`\|`FAILED`), provider, providerPayoutId, scheduledAt, paidAt | |
| **Refund** | paymentId (fk), amount, reason, status, initiatedBy (fk User admin) | |
| **Review** | serviceId (fk), authorId (fk User), targetId (fk User), ratingOverall (1–5), ratingQuality, ratingPunctuality, ratingCleanliness, ratingCommunication, ratingValue, comment?, isFlagged, flagReason? | único por (serviceId, authorId); solo si Service.status = PAID/CONFIRMED |
| **Chat** | serviceId (fk, unique) | 1–N ChatMessage |
| **ChatMessage** | chatId (fk), senderId? (fk User, null=sistema), type (`TEXT`\|`PHOTO`\|`SYSTEM`), content?, mediaUrl?, readAt? | |
| **ServicePhoto** | serviceId (fk), uploadedBy (fk User), type (`BEFORE`\|`AFTER`), url | usado en disputas/garantías |
| **FavoriteProvider** | customerId (fk), providerId (fk) | unique (customerId, providerId) |
| **Dispute** | serviceId (fk), openedBy (fk User), reason, description, status (`OPEN`\|`IN_REVIEW`\|`RESOLVED`\|`REJECTED`), resolution?, resolvedBy? (fk User admin), resolvedAt?, refundAmount? | evidencia = ServicePhoto + ChatMessage del mismo Service |
| **WarrantyClaim** | serviceId (fk), customerId (fk), description, mediaUrls[], status (`OPEN`\|`IN_REVIEW`\|`APPROVED_REPAIR`\|`APPROVED_REFUND_PARTIAL`\|`APPROVED_REFUND_FULL`\|`REJECTED`\|`CLOSED`), assignedProviderId? (fk), resolvedBy?, resolvedAt?, resolutionNotes? | |
| **Cancellation** | serviceId (fk), cancelledBy (fk User), role (`CUSTOMER`\|`PROVIDER`\|`SYSTEM`), reason, feeApplied (bool), feeAmount? | |
| **Notification** | userId (fk), type, title, body, data (json), channel (`PUSH`\|`EMAIL`\|`SMS`), status (`PENDING`\|`SENT`\|`FAILED`\|`READ`), sentAt?, readAt? | |
| **DeviceToken** | userId (fk), token, platform (`IOS`\|`ANDROID`) | para push |
| **SafetyReport** | serviceId? (fk), reporterId (fk User), reportedUserId (fk User), type (`SAFETY`\|`BEHAVIOR`\|`ADDRESS`\|`THREAT`\|`OTHER`), description, status (`OPEN`\|`REVIEWED`\|`ACTIONED`) | |
| **BlockedUser** | providerId (fk), blockedCustomerId (fk), reason, status (`PENDING_REVIEW`\|`APPROVED`\|`REJECTED`) | bloqueo sujeto a revisión admin |
| **AdminAction** | adminId (fk User), actionType, targetType, targetId, notes | |
| **AuditLog** | actorId? (fk User), actorType, action, entityType, entityId, oldValue (json), newValue (json), ipAddress | inmutable, sin updatedAt |
| **TermsAcceptance** | userId (fk), documentType (`TERMS`\|`PRIVACY`), version, acceptedAt | |

**Índices clave:** `Service(customerId, status)`, `Service(providerId, status)`, `ProviderProfile(isOnline, currentGeo)` como índice GIST (PostGIS) para queries de distancia, `ProviderSkill(categoryId, subcategoryId)`, `Notification(userId, status)`, únicos en `User.email`, `User.phone`, `Category.slug`, `(Subcategory.categoryId, slug)`.

---

## 7. Máquina de estados del servicio

```
REQUESTED
   │ (system: crea ServiceRequest → busca profesionales)
   ▼
SEARCHING_PROVIDER ──(sin respuesta / timeout / cliente cancela)──► CANCELLED
   │ (profesional acepta oferta)
   ▼
PROVIDER_ASSIGNED ──(cliente o profesional cancela)──► CANCELLED
   │ (profesional: "En camino")
   ▼
PROVIDER_EN_ROUTE ──(cancelación)──► CANCELLED
   │ (profesional: "Llegué")
   ▼
PROVIDER_ARRIVED
   │
   ├─ si subcategory.requiresQuoteApproval = true:
   │     ▼
   │   WAITING_FOR_QUOTE_APPROVAL ──(cliente rechaza)──► CANCELLED (o vuelve a WAITING con nueva cotización)
   │     │ (cliente aprueba)
   │     ▼
   │   QUOTE_APPROVED
   │     │
   ├─ si no requiere aprobación: pasa directo desde PROVIDER_ARRIVED
   │
   ▼ (profesional introduce PIN de 4 dígitos que ve el cliente)
IN_PROGRESS
   │ (profesional sube fotos AFTER y marca terminado)
   ▼
COMPLETED_BY_PROVIDER
   │ (cliente confirma) ó (auto-confirmación por sistema tras N horas sin respuesta)
   ▼
CONFIRMED_BY_CUSTOMER
   │ (sistema: captura el pago)
   ▼
PAID ──► puede abrir, dentro de ventana de tiempo:
           ├─ DISPUTED (reclamo sobre el cobro/servicio, requiere admin)
           └─ WARRANTY_CLAIM (falla post-servicio, dentro de warrantyExpiresAt)

DISPUTED / WARRANTY_CLAIM se resuelven por Admin con: reparación (nuevo Service ligado), reembolso parcial, reembolso total, o rechazo justificado → estado final REFUNDED o vuelta a PAID/CLOSED.
```

Reglas duras validadas en backend (no solo en UI):

- No se puede pasar a `IN_PROGRESS` sin PIN verificado.
- No se puede pasar a `IN_PROGRESS` si `requiresQuoteApproval = true` y no hay `Quote.status = APPROVED`.
- Toda transición se registra en `ServiceStatusHistory` con el actor que la ejecutó.
- Transiciones inválidas (ej. `PROVIDER_ARRIVED → PAID`) son rechazadas por un guard de la máquina de estados en el módulo `services`.
- `CANCELLED` solo es alcanzable antes de `IN_PROGRESS` sin intervención de un admin; cancelar después de `IN_PROGRESS` requiere revisión (queda como `DISPUTED`).

---

## 8. Motor de matching (diseño modular)

Servicio independiente (`modules/matching`) que, para una `ServiceRequest` urgente, calcula un score por profesional candidato:

```
score = w1·distanceScore + w2·ratingScore + w3·reliabilityScore
      + w4·experienceScore + w5·availabilityScore
```

- `distanceScore`: normalizado por distancia real vía PostGIS (`ST_DWithin` / `ST_Distance`), no línea recta pura si hay ETA disponible.
- `ratingScore`: `ratingAvg` normalizado 0–1.
- `reliabilityScore`: combina `acceptanceRate`, `cancellationRate` y `punctualityScore`.
- `experienceScore`: años de experiencia en esa `Subcategory` + trabajos completados en ella.
- `availabilityScore`: penaliza si el profesional ya tiene un servicio activo o si está cerca del límite de radio de su `ProviderZone`.

Los pesos (`w1..w5`) viven en una tabla de configuración (`MatchingWeightConfig`, editable desde el panel admin) — no hardcodeados. El motor ofrece el trabajo a los top-N candidatos en oleadas (primero top 3, si nadie acepta en X segundos amplía el radio/oleada), vía BullMQ. Diseño pensado para que cambiar el algoritmo (ej. agregar señales de fraude) no afecte al resto del sistema — es un puerto/adaptador más.

---

## 9. API — endpoints principales

Prefijo base `/api/v1`. Todos los endpoints autenticados requieren `Authorization: Bearer <accessToken>`; los de admin además requieren `role=ADMIN`.

**Auth**
```
POST   /auth/register              (cliente; profesional usa /providers/onboarding)
POST   /auth/otp/request
POST   /auth/otp/verify
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /auth/me
```

**Catálogo**
```
GET    /categories
GET    /categories/:id/subcategories
```

**Cliente — direcciones y solicitudes**
```
GET/POST/PATCH/DELETE  /addresses
POST   /service-requests
GET    /service-requests/:id
POST   /service-requests/:id/media
GET    /providers/:id                       (perfil público)
POST   /favorites/:providerId
DELETE /favorites/:providerId
GET    /services                             (mis servicios, filtrable por estado)
GET    /services/:id
POST   /services/:id/quotes/:quoteId/approve
POST   /services/:id/quotes/:quoteId/reject
GET    /services/:id/pin
POST   /services/:id/confirm-completion
POST   /services/:id/cancel
POST   /services/:id/reviews
POST   /services/:id/warranty-claims
POST   /services/:id/reports
POST   /services/:id/share                   (genera link de "compartir servicio")
GET    /services/:id/chat/messages
POST   /services/:id/chat/messages
```

**Profesional**
```
POST   /providers/onboarding                 (multi-step, ver Fase 3)
POST   /providers/me/verifications
PATCH  /providers/me/availability             { isOnline }
PATCH  /providers/me/location                 { lat, lng }
GET    /providers/me/job-offers
POST   /job-offers/:id/accept
POST   /job-offers/:id/reject
POST   /services/:id/en-route
POST   /services/:id/arrived
POST   /services/:id/quotes
POST   /services/:id/pin/verify
POST   /services/:id/start
POST   /services/:id/complete
POST   /services/:id/photos                   { type: BEFORE|AFTER }
GET    /providers/me/earnings
```

**Admin**
```
GET/PATCH   /admin/users
GET         /admin/providers/pending
POST        /admin/providers/:id/approve
POST        /admin/providers/:id/reject
POST        /admin/providers/:id/suspend
GET/POST/PATCH  /admin/categories
GET/POST/PATCH  /admin/subcategories
GET/POST/PATCH  /admin/commission-rules
GET/POST/PATCH  /admin/warranty-policies
GET/PATCH   /admin/matching-weights
GET         /admin/services
GET/PATCH   /admin/disputes/:id
GET/PATCH   /admin/warranty-claims/:id
GET         /admin/safety-reports
GET         /admin/metrics/overview
GET         /admin/audit-logs
```

**Webhooks**
```
POST   /webhooks/payments/:provider
```

**Realtime (WebSocket, namespaces)**
```
/ws/chat              → mensajes por serviceId (room)
/ws/tracking          → ubicación del profesional en vivo (room = serviceId)
/ws/job-offers        → ofertas de trabajo push al profesional online
/ws/notifications     → notificaciones in-app en tiempo real
```

---

## 10. Roles y permisos (RBAC)

| Recurso / acción | Cliente | Profesional | Admin |
|---|---|---|---|
| Crear ServiceRequest | ✔ (propio) | ✘ | ✔ |
| Ver Service | ✔ (solo si `customerId = self`) | ✔ (solo si `providerId = self`) | ✔ (todos) |
| Aceptar/rechazar oferta | ✘ | ✔ (propio) | ✘ |
| Generar Quote | ✘ | ✔ (servicio propio) | ✘ |
| Aprobar/rechazar Quote | ✔ (propio) | ✘ | ✔ (soporte) |
| Verificar PIN | ✘ | ✔ (propio) | ✘ |
| Subir ServicePhoto | ✘ | ✔ (propio) | lectura: ✔ |
| Confirmar finalización | ✔ (propio) | ✘ | ✔ (override) |
| Crear Review | ✔ (propio, solo tras `PAID`) | ✔ (propio, opcional sobre cliente) | ✘ |
| Abrir Dispute / WarrantyClaim | ✔/✔ (propio) | ✔ (dispute, propio) | ✘ (solo resuelve) |
| Resolver Dispute/WarrantyClaim | ✘ | ✘ | ✔ |
| CRUD Category/Subcategory | ✘ | ✘ | ✔ |
| CRUD CommissionRule | ✘ | ✘ | ✔ |
| Aprobar/rechazar/suspender profesional | ✘ | ✘ | ✔ |
| Ver ProviderVerification (documentos) | ✘ | ✔ (propios) | ✔ (autorizado) |
| Ver AuditLog | ✘ | ✘ | ✔ (solo `SUPERADMIN`) |
| Emitir reembolso | ✘ | ✘ | ✔ (`FINANCE`/`SUPERADMIN`) |

Implementación: `RolesGuard` global + `@Roles()` decorator para el rol base, y un `OwnershipGuard`/`CaslAbilityGuard` por recurso para reglas "solo lo propio". `adminRole` permite permisos admin granulares desde el MVP (aunque en V1 todo admin puede ser `SUPERADMIN` por defecto) sin rediseñar el esquema después.

---

## 11. Variables de entorno

```
# App
NODE_ENV=
PORT=
APP_URL=

# Base de datos
DATABASE_URL=                      # postgres://... (con PostGIS habilitado)

# Redis
REDIS_URL=

# Auth
JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=30d

# Storage (S3-compatible)
S3_REGION=
S3_BUCKET_PUBLIC=
S3_BUCKET_PRIVATE=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
S3_SIGNED_URL_TTL_SECONDS=300

# Mapas
GOOGLE_MAPS_API_KEY=

# Pagos (dejar vacío = usa PaymentGateway mock automáticamente)
PAYMENTS_PROVIDER=mock             # mock | stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_CONNECT_CLIENT_ID=

# SMS / OTP (dejar vacío = usa SmsProvider mock, imprime OTP en logs)
SMS_PROVIDER=mock                  # mock | twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_FROM_NUMBER=

# Email (dejar vacío = usa EmailProvider mock)
EMAIL_PROVIDER=mock                # mock | resend | sendgrid
EMAIL_API_KEY=
EMAIL_FROM=

# Push notifications
EXPO_ACCESS_TOKEN=

# Observabilidad (opcional)
SENTRY_DSN=

# Mobile (Expo, prefijo EXPO_PUBLIC_)
EXPO_PUBLIC_API_URL=
EXPO_PUBLIC_WS_URL=
EXPO_PUBLIC_GOOGLE_MAPS_API_KEY=
```

> Todo lo marcado "dejar vacío = mock" funciona sin credenciales reales en desarrollo. En la Fase correspondiente indicaré exactamente dónde pegar cada clave cuando la tengas.

---

## 12. Pagos — diseño de la abstracción (sin inventar integraciones)

```typescript
interface PaymentGateway {
  createPaymentIntent(input: { amountCents: number; currency: 'MXN'; serviceId: string }): Promise<PaymentIntentResult>;
  capture(paymentIntentId: string): Promise<CaptureResult>;
  refund(paymentIntentId: string, amountCents?: number): Promise<RefundResult>;
  createPayoutAccount(provider: ProviderProfile): Promise<PayoutAccountResult>;
  payout(providerAccountId: string, amountCents: number): Promise<PayoutResult>;
}
```

- `MockPaymentGateway`: implementación real y funcional (no pseudocódigo) que simula estados (`AUTHORIZED → CAPTURED`), permite testear todo el flujo de dinero end-to-end sin proveedor externo.
- Adaptador recomendado para México: **Stripe** (soporta MXN, tarjetas mexicanas, y Stripe Connect resuelve el payout al profesional). Alternativa con métodos locales (OXXO/SPEI) para Fase 2: **Conekta** u **OpenPay**, implementando la misma interfaz.
- El dominio (`services`, `payments`) nunca importa el SDK del proveedor directamente — solo la interfaz. Cuando tengas las credenciales, se agrega el adaptador y se cambia `PAYMENTS_PROVIDER=stripe`; cero cambios en lógica de negocio.

---

## 13. Qué incluye el MVP

Cliente: registro/login, categorías/subcategorías, solicitud (urgente/programado/cotización) con fotos/video, matching, perfil de profesional, chat, presupuesto + aprobación, PIN, todos los estados del servicio, pago (mock o Stripe si hay credenciales), review, historial, favoritos, compartir servicio, reportar/ayuda.

Profesional: onboarding completo con verificación documental, toggle online/offline, ofertas de trabajo, aceptar/rechazar, navegación (en camino/llegué), presupuesto, PIN, fotos antes/después, finalizar, historial de ingresos.

Admin: usuarios, verificación de profesionales, servicios (vista completa: estado/ubicación/presupuesto/fotos/chat/pago), categorías/subcategorías, comisiones, garantías configurables, disputas, reportes de seguridad, reviews, métricas básicas, audit log.

Todo lo anterior queda **funcional de extremo a extremo**, incluso las integraciones externas (pagos, SMS, mapas) mediante mocks reales o el proveedor si se proveen credenciales.

## 14. Qué queda preparado pero no activo (Fase 2+)

IA para clasificar problema por foto/texto, estimación automática de precio, suscripciones, fidelidad, cuentas empresariales/condominios, múltiples ciudades, promociones complejas, seguro integrado, referidos avanzados, separación en dos apps móviles, microservicios. La arquitectura (módulos desacoplados, interfaces de proveedores externos, tabla de verificaciones extensible, categorías/subcategorías dinámicas, matching con pesos configurables) ya contempla estos puntos: agregarlos no debería requerir rediseño, solo nuevos módulos/adaptadores.

---

**Siguiente paso:** con tu aprobación de este plan, continúo con **Fase 2 — Backend** (estructura NestJS real, `schema.prisma` completo, migraciones, módulos base, auth).
