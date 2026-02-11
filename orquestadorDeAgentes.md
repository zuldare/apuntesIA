# Orquestador de Agentes MAT

**Versión**: 1.0.0
**Fecha**: 2025-12-24
**Autor**: Jaime Hernandez

Este documento describe cómo organizar y orquestar los agentes disponibles para el ecosistema MAT.

---

## Inventario de Agentes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGENTES MAT DISPONIBLES                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  mat-code-reviewer  │  │ mat-dependency-sync │  │  mat-aws-reviewer   │  │
│  │  ─────────────────  │  │  ─────────────────  │  │  ─────────────────  │  │
│  │  Revisión de código │  │  Sync de versiones  │  │  Auditoría AWS      │  │
│  │  Java/Spring Boot   │  │  entre servicios    │  │  S3/SQS/Lambda...   │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │ mat-security-scanner│  │mat-migration-orch.  │  │mat-cross-service    │  │
│  │  ─────────────────  │  │  ─────────────────  │  │  ─────────────────  │  │
│  │  OAuth2/SAML/JWT    │  │  Migraciones Spring │  │  Contratos Feign    │  │
│  │  Spring Security    │  │  Boot coordinadas   │  │  APIs compartidas   │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                             │
│  ┌─────────────────────┐                                                    │
│  │mat-bdd-test-generator│                                                   │
│  │  ─────────────────  │                                                    │
│  │  Tests Cucumber     │                                                    │
│  │  Gherkin con mocks  │                                                    │
│  └─────────────────────┘                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Clasificación por Propósito

```
                              AGENTES MAT
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
    ┌───────────┐           ┌───────────┐           ┌───────────┐
    │ CALIDAD   │           │ SEGURIDAD │           │ARQUITECTURA│
    │ DE CÓDIGO │           │           │           │           │
    └─────┬─────┘           └─────┬─────┘           └─────┬─────┘
          │                       │                       │
          ▼                       ▼                       ▼
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │mat-code-     │        │mat-security- │        │mat-cross-    │
   │reviewer      │        │scanner       │        │service-      │
   └──────────────┘        └──────────────┘        │analyzer      │
   ┌──────────────┐        ┌──────────────┐        └──────────────┘
   │mat-bdd-test- │        │mat-aws-      │        ┌──────────────┐
   │generator     │        │reviewer      │        │mat-dependency│
   └──────────────┘        └──────────────┘        │-sync         │
                                                   └──────────────┘
                                                   ┌──────────────┐
                                                   │mat-migration-│
                                                   │orchestrator  │
                                                   └──────────────┘
```

---

## Flujos de Orquestación

### Flujo 1: Desarrollo de Nueva Feature

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FLUJO: DESARROLLO DE NUEVA FEATURE                       │
└─────────────────────────────────────────────────────────────────────────────┘

     INICIO
        │
        ▼
   ┌─────────┐
   │ Escribir│
   │ código  │
   └────┬────┘
        │
        ▼
   ┌─────────────────────┐
   │  mat-code-reviewer  │◄─────── Revisión automática post-cambio
   │  (PROACTIVO)        │
   └──────────┬──────────┘
              │
      ┌───────┴───────┐
      │               │
      ▼               ▼
  ┌───────┐      ┌────────┐
  │ PASS  │      │ ISSUES │──────► Corregir y repetir
  └───┬───┘      └────────┘
      │
      ▼
   ┌─────────────────────┐
   │mat-bdd-test-generator│◄────── Generar tests de comportamiento
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────┐
   │ Ejecutar tests      │
   │ ./mvnw test         │
   └──────────┬──────────┘
              │
      ┌───────┴───────┐
      │               │
      ▼               ▼
  ┌───────┐      ┌────────┐
  │ PASS  │      │ FAIL   │──────► Corregir y repetir
  └───┬───┘      └────────┘
      │
      ▼
   ┌─────────────────────┐
   │mat-cross-service-   │◄────── Solo si hay cambios en APIs/DTOs
   │analyzer (OPCIONAL)  │
   └──────────┬──────────┘
              │
              ▼
         COMMIT & PR
```

### Flujo 2: Pre-Merge / Code Review

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FLUJO: PRE-MERGE / CODE REVIEW                         │
└─────────────────────────────────────────────────────────────────────────────┘

        PR ABIERTO
            │
            ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                    EJECUCIÓN EN PARALELO                       │
   │                                                                │
   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
   │  │mat-code-reviewer│  │mat-security-    │  │mat-aws-reviewer │ │
   │  │                 │  │scanner          │  │(si hay cambios  │ │
   │  │ Calidad código  │  │                 │  │ en config AWS)  │ │
   │  │ Patrones Spring │  │ Vulnerabilidades│  │                 │ │
   │  │ Performance     │  │ OWASP Top 10    │  │ S3/SQS/Lambda   │ │
   │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
   │           │                    │                    │          │
   └───────────┼────────────────────┼────────────────────┼──────────┘
               │                    │                    │
               ▼                    ▼                    ▼
        ┌──────────────────────────────────────────────────────┐
        │              CONSOLIDAR REPORTES                      │
        │                                                       │
        │  📋 Code Review Report                               │
        │  🔒 Security Scan Report                             │
        │  ☁️  AWS Configuration Report                         │
        └──────────────────────────────────────────────────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │  DECISIÓN DE MERGE     │
                 │                        │
                 │  🟢 Sin issues críticos│──► MERGE
                 │  🟡 Issues menores     │──► MERGE + TODO
                 │  🔴 Issues críticos    │──► BLOQUEAR
                 └────────────────────────┘
```

### Flujo 3: Migración de Spring Boot

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FLUJO: MIGRACIÓN SPRING BOOT                             │
└─────────────────────────────────────────────────────────────────────────────┘

                         INICIO MIGRACIÓN
                               │
                               ▼
                  ┌────────────────────────┐
                  │  mat-dependency-sync   │◄── Estado actual de versiones
                  │                        │
                  │  Detectar drift entre  │
                  │  servicios             │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ mat-migration-         │◄── Plan de migración
                  │ orchestrator           │
                  │                        │
                  │  - Breaking changes    │
                  │  - Orden de servicios  │
                  │  - Timeline            │
                  └───────────┬────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
     │  Servicio 1 │  │  Servicio 2 │  │  Servicio N │
     │  (pilot)    │  │             │  │             │
     └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
            │                │                │
            │    ┌───────────┴────────────────┘
            │    │
            ▼    ▼
   ┌────────────────────────┐
   │  POR CADA SERVICIO:    │
   │                        │
   │  1. Actualizar POM     │
   │  2. Compilar           │
   │  3. Ejecutar tests     │
   │  4. mat-code-reviewer  │◄── Detectar deprecations
   │  5. mat-security-scan  │◄── Verificar seguridad
   │  6. Deploy a DEV       │
   └───────────┬────────────┘
               │
               ▼
   ┌────────────────────────┐
   │mat-cross-service-      │◄── Verificar compatibilidad
   │analyzer                │    entre servicios migrados
   │                        │
   │  - Contratos Feign OK? │
   │  - DTOs compatibles?   │
   │  - Auth funcionando?   │
   └───────────┬────────────┘
               │
               ▼
   ┌────────────────────────┐
   │  mat-dependency-sync   │◄── Actualizar documento
   │                        │    con nuevo estado
   │  Verificar todas las   │
   │  versiones alineadas   │
   └───────────┬────────────┘
               │
               ▼
          FIN MIGRACIÓN
```

### Flujo 4: Auditoría de Seguridad

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FLUJO: AUDITORÍA DE SEGURIDAD                          │
└─────────────────────────────────────────────────────────────────────────────┘

                    TRIGGER: Scheduled / Pre-Release
                               │
                               ▼
   ┌───────────────────────────────────────────────────────────────────────┐
   │                        ESCANEO PARALELO                               │
   │                                                                       │
   │    auth-service    notification    audit       admin      param      │
   │         │              │            │           │           │        │
   │         ▼              ▼            ▼           ▼           ▼        │
   │    ┌─────────┐    ┌─────────┐  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
   │    │security │    │security │  │security │ │security │ │security │   │
   │    │-scanner │    │-scanner │  │-scanner │ │-scanner │ │-scanner │   │
   │    └────┬────┘    └────┬────┘  └────┬────┘ └────┬────┘ └────┬────┘   │
   │         │              │            │           │           │        │
   └─────────┼──────────────┼────────────┼───────────┼───────────┼────────┘
             │              │            │           │           │
             └──────────────┴─────┬──────┴───────────┴───────────┘
                                  │
                                  ▼
                     ┌────────────────────────┐
                     │  CONSOLIDAR REPORTES   │
                     │                        │
                     │  - Credenciales        │
                     │  - OWASP Top 10        │
                     │  - CVEs en deps        │
                     │  - Config errónea      │
                     └───────────┬────────────┘
                                 │
                                 ▼
                     ┌────────────────────────┐
                     │   mat-aws-reviewer     │◄── Complementar con
                     │                        │    auditoría AWS
                     │   - IAM policies       │
                     │   - S3 buckets         │
                     │   - Secrets Manager    │
                     └───────────┬────────────┘
                                 │
                                 ▼
                     ┌────────────────────────┐
                     │  SECURITY DASHBOARD    │
                     │                        │
                     │  🔴 Critical: X        │
                     │  🟡 High: Y            │
                     │  🟢 Medium: Z          │
                     └────────────────────────┘
```

### Flujo 5: Generación de Tests BDD

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FLUJO: GENERACIÓN DE TESTS BDD                         │
└─────────────────────────────────────────────────────────────────────────────┘

                    TRIGGER: Nueva funcionalidad / Sprint Planning
                               │
                               ▼
                  ┌────────────────────────┐
                  │  Identificar Servicio  │
                  │  a testear             │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │mat-bdd-test-generator  │
                  │                        │
                  │  1. Analizar Service   │
                  │  2. Identificar métodos│
                  │  3. Seleccionar        │
                  │     patrones           │
                  └───────────┬────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                     PATRONES APLICADOS                               │
   │                                                                      │
   │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐│
   │  │ LISTA        │ │ CRUD         │ │ ERRORES      │ │ FINANCIERO   ││
   │  │              │ │              │ │              │ │              ││
   │  │ - Empty      │ │ - Create OK  │ │ - Not Found  │ │ - BigDecimal ││
   │  │ - One        │ │ - Read OK    │ │ - Validation │ │ - Precisión  ││
   │  │ - Many       │ │ - Update OK  │ │ - Duplicate  │ │ - Redondeo   ││
   │  │ - Paginated  │ │ - Delete OK  │ │ - Conflict   │ │              ││
   │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘│
   │                                                                      │
   └──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │  ARCHIVOS GENERADOS    │
                  │                        │
                  │  📁 features/          │
                  │    └── service.feature │
                  │  📁 steps/             │
                  │    └── ServiceSteps.java│
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │  ./mvnw test           │
                  │  -Dcucumber.filter.tags│
                  └────────────────────────┘
```

---

## Matriz de Decisión: ¿Qué Agente Usar?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MATRIZ DE DECISIÓN                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ¿Qué estás haciendo?                          Agente(s) a usar             │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                             │
│  📝 Escribí código nuevo                       → mat-code-reviewer          │
│                                                                             │
│  🔄 Actualicé un endpoint/DTO                  → mat-cross-service-analyzer │
│                                                                             │
│  ☁️  Cambié configuración AWS                   → mat-aws-reviewer          │
│                                                                             │
│  🔐 Toqué seguridad/auth                       → mat-security-scanner       │
│                                                                             │
│  📦 Quiero actualizar dependencias             → mat-dependency-sync        │
│                                                                             │
│  🚀 Migración de Spring Boot                   → mat-migration-orchestrator │
│                                                → mat-dependency-sync        │
│                                                → mat-cross-service-analyzer │
│                                                                             │
│  🧪 Necesito tests para un servicio            → mat-bdd-test-generator     │
│                                                                             │
│  📋 Code Review de PR                          → mat-code-reviewer          │
│                                                → mat-security-scanner       │
│                                                                             │
│  🔍 Auditoría completa pre-release             → TODOS                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prioridad de Ejecución

Cuando se ejecutan múltiples agentes, este es el orden recomendado:

```
                           PRIORIDAD DE EJECUCIÓN
                                    │
     ┌──────────────────────────────┼──────────────────────────────┐
     │                              │                              │
     ▼                              ▼                              ▼
┌─────────┐                   ┌─────────┐                    ┌─────────┐
│PRIORIDAD│                   │PRIORIDAD│                    │PRIORIDAD│
│   1     │                   │   2     │                    │   3     │
│ CRÍTICO │                   │  ALTO   │                    │ NORMAL  │
└────┬────┘                   └────┬────┘                    └────┬────┘
     │                              │                              │
     ▼                              ▼                              ▼
┌─────────────┐              ┌─────────────┐               ┌─────────────┐
│mat-security-│              │mat-code-    │               │mat-bdd-test-│
│scanner      │              │reviewer     │               │generator    │
│             │              │             │               │             │
│"Primero     │              │"Después     │               │"Tests       │
│ seguridad"  │              │ calidad"    │               │ al final"   │
└─────────────┘              └─────────────┘               └─────────────┘
                             ┌─────────────┐               ┌─────────────┐
                             │mat-cross-   │               │mat-dependency│
                             │service-     │               │-sync        │
                             │analyzer     │               │             │
                             └─────────────┘               └─────────────┘
                             ┌─────────────┐
                             │mat-aws-     │
                             │reviewer     │
                             └─────────────┘
```

---

## Integración con CI/CD

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PIPELINE CI/CD                                      │
└─────────────────────────────────────────────────────────────────────────────┘

   PR Opened
      │
      ▼
┌───────────────┐
│   BUILD       │
│   ./mvnw      │
│   compile     │
└───────┬───────┘
        │
        ▼
┌───────────────┐     ┌─────────────────────────────────────────────────────┐
│   TEST        │     │  PARALLEL QUALITY GATES                             │
│   ./mvnw test │     │                                                     │
└───────┬───────┘     │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
        │             │  │mat-security-│ │mat-code-    │ │mat-aws-     │   │
        ├────────────►│  │scanner      │ │reviewer     │ │reviewer     │   │
        │             │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │
        │             │         │               │               │          │
        │             │         ▼               ▼               ▼          │
        │             │     ┌───────────────────────────────────────┐      │
        │             │     │         QUALITY REPORT                │      │
        │             │     │                                       │      │
        │             │     │  Security: PASS/FAIL                  │      │
        │             │     │  Code Quality: PASS/FAIL              │      │
        │             │     │  AWS Config: PASS/FAIL                │      │
        │             │     └───────────────────────────────────────┘      │
        │             └─────────────────────────────────────────────────────┘
        │                                    │
        ▼                                    ▼
┌───────────────┐                   ┌───────────────┐
│   MERGE       │◄──────────────────│  ALL PASS?    │
│   DECISION    │                   │  🟢 Yes       │
└───────────────┘                   └───────────────┘
```

---

## Comandos Rápidos

```bash
# Invocar agente de code review en servicio actual
# (desde el directorio del servicio)
claude "Ejecuta mat-code-reviewer en este servicio"

# Sincronizar dependencias entre todos los servicios
claude "Ejecuta mat-dependency-sync para C:\githubMat"

# Auditoría de seguridad completa
claude "Ejecuta mat-security-scanner en back-auth-service"

# Generar tests BDD para un servicio
claude "Genera tests BDD para ScenarioService usando mat-bdd-test-generator"

# Verificar compatibilidad antes de release
claude "Ejecuta mat-cross-service-analyzer entre param-service y reports-service"

# Planificar migración
claude "Usa mat-migration-orchestrator para planificar migración a Spring Boot 3.4"
```

---

## Diagrama de Dependencias entre Agentes

```
                        ┌─────────────────────────┐
                        │ mat-migration-          │
                        │ orchestrator            │
                        │                         │
                        │ "Orquesta migraciones"  │
                        └───────────┬─────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
          ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
          │mat-dependency│  │mat-cross-   │  │mat-security-│
          │-sync        │  │service-     │  │scanner      │
          │             │  │analyzer     │  │             │
          │"Versiones"  │  │"Contratos"  │  │"Seguridad"  │
          └─────────────┘  └─────────────┘  └──────┬──────┘
                                                   │
                                                   ▼
                                           ┌─────────────┐
                                           │mat-aws-     │
                                           │reviewer     │
                                           │             │
                                           │"AWS Config" │
                                           └─────────────┘

                        ┌─────────────────────────┐
                        │   mat-code-reviewer     │
                        │                         │
                        │   "Calidad de código"   │
                        │   (INDEPENDIENTE)       │
                        └───────────┬─────────────┘
                                    │
                                    ▼
                        ┌─────────────────────────┐
                        │ mat-bdd-test-generator  │
                        │                         │
                        │ "Genera tests después   │
                        │  de revisar código"     │
                        └─────────────────────────┘
```

---

## Checklist de Uso por Escenario

### Antes de Commit
- [ ] `mat-code-reviewer` - Revisar cambios locales

### Antes de Crear PR
- [ ] `mat-code-reviewer` - Revisión completa
- [ ] `mat-security-scanner` - Si hay cambios de seguridad
- [ ] `mat-cross-service-analyzer` - Si hay cambios en APIs

### Antes de Merge a Develop
- [ ] `mat-code-reviewer` - Revisión final
- [ ] `mat-security-scanner` - Escaneo de seguridad
- [ ] `mat-bdd-test-generator` - Verificar cobertura de tests

### Antes de Release
- [ ] `mat-dependency-sync` - Verificar versiones alineadas
- [ ] `mat-security-scanner` - Auditoría completa
- [ ] `mat-aws-reviewer` - Verificar configuraciones AWS
- [ ] `mat-cross-service-analyzer` - Verificar compatibilidad

### Migración de Versión
- [ ] `mat-dependency-sync` - Estado inicial
- [ ] `mat-migration-orchestrator` - Plan de migración
- [ ] `mat-code-reviewer` - Por cada servicio
- [ ] `mat-security-scanner` - Por cada servicio
- [ ] `mat-cross-service-analyzer` - Verificar integraciones
- [ ] `mat-dependency-sync` - Estado final

---

*Documento generado automáticamente. Actualizar después de añadir nuevos agentes.*
