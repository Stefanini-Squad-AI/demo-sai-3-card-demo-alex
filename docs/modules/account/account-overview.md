# 💳 ACCOUNT - Módulo de Cuentas

**Módulo ID**: `ACCOUNT`  
**Versión**: 1.0  
**Última actualización**: 2026-01-26  
**Propósito**: Consulta y actualización guiada del ciclo de vida de cuentas de clientes, con foco en cumplimiento (enmascarado de datos), validaciones en tiempo real y experiencias de back-office basadas en Material-UI.

---

## 📋 Descripción General

El módulo ACCOUNT empuja toda la lógica de front-end y back-end necesaria para que los equipos operativos puedan buscar una cuenta por su identificador, analizar su estado financiero y aplicar cambios autorizados. La experiencia se divide en dos pantallas principales (view y update) que comparten componentes, hooks y servicios de validación para garantizar consistencia y reutilización.

### Responsabilidades clave
- Validar Account ID (exactamente 11 dígitos numéricos, no todos ceros) antes de consultar.
- Inicializar la pantalla con metadata (`GET /account-view/initialize`) y refrescar datos con estados de carga/errores centralizados (`useMutation`).
- Mostrar tarjetas de información financiero, cliente y contacto con características como toggle de datos sensibles y chips de estado.
- Habilitar un modo de edición controlado con `Edit Mode`, detección automática de cambios y confirmación antes de enviar actualizaciones.
- Aplicar validaciones específicas (Y/N para activeStatus, formato ZIP 5-4, campos numéricos, etc.).
- Mantener autenticación por rol leyendo `localStorage.userRole` y redirigiendo a `/login` o menús principales según corresponde.
- Habilitar accesos rápidos ya sea vía botones o atajos de teclado (`F3`, `Escape`, `F5`, `F12`) y mostrar cuentas de prueba en entorno DEV.

---

## 🏗️ Arquitectura del Módulo

### Componentes clave
1. **`AccountViewPage.tsx`** – Página envuelta en `React Router`: valida la sesión (`localStorage.userRole`), inicializa la pantalla (`initializeScreen`), expone `searchAccount` al componente de UI y redirige al menú correcto cuando se cierra.
2. **`AccountViewScreen.tsx`** – Vista pesada en MUI donde se encuentra:
   - Formulario de búsqueda con input limitado a 11 dígitos y validación inline.
   - Botones `Search`, `Show Test Accounts` (solo en `import.meta.env.DEV`), `Show/Hide Sensitive Data`.
   - Tarjetas (`Card`, `Grid`, `Stack`) para Account Information, Financial Information, Customer Overview, Contact & Personal Info con iconografía de `@mui/icons-material`.
   - Alertas para errores, chips de estado (Activo/Inactivo), y `LoadingSpinner` para UX de carga.
3. **`useAccountView.ts`** – Hook personalizado sobre `useMutation` que envía:
   - `GET /account-view?accountId={id}` (el ID se parsea y se padStart a 11 dígitos).
   - `GET /account-view/initialize`.
   - `clearData` y `reset` para limpiar el estado entre búsquedas.
4. **`AccountUpdatePage.tsx`** – Similar a la página de vista: valida sesión, limpia datos (`clearData`) y entrega los handlers (`updateLocalData`, `resetForm`, `updateAccount`).
5. **`AccountUpdateScreen.tsx`** – Pantalla de edición que ofrece:
   - Formulario de búsqueda igual al viewer.
   - Toggle `Edit Mode` con `Switch`.
   - Detección de cambios (`hasChanges`) comparando JSON del estado original.
   - Validaciones locales (ZIP, números, estado activo) con mensajes en `TextField.helperText`.
   - Dialogo de confirmación antes de enviar (`Dialog`, `DialogActions`).
   - Atajos de teclado: `F3`/`Escape` para salir, `F5` para guardar, `F12` para resetear.
6. **`useAccountUpdate.ts`** – Hook que:
   - Busca vía `GET /accounts/{accountId}` y guarda `accountData` + `originalData`.
   - Actualiza con `PUT /accounts/{accountId}`.
   - Expone `updateLocalData`, `resetForm`, `clearData`, banderas de `loading`/`error` y `hasChanges`.

### Servicios auxiliares
- `SystemHeader` y `LoadingSpinner` garantizan consistencia visual y comunicación de estado.
- `apiClient` centraliza la baseURL y headers (reutilizado por todos los hooks del módulo).
- `useMutation` de `app/hooks/useApi.ts` maneja retries, cancelaciones y respuestas tanto de MSW (`ApiResponse`) como del backend real.

---

## 🔗 APIs Documentadas

1. **GET `/api/account-view?accountId={id}`**  
   Request obligatoriamente numérico y se convierte a `parseInt` + `padStart(11)` en el frontend. Respuesta estándar (`AccountViewResponse`) incluye fecha actual, cuenta, cliente y mensajes.

2. **GET `/api/account-view/initialize`**  
   Devuelve metadata inicial (fecha, transactionId) que el hook `initializeScreen` consume para mostrar un estado pre-cargado al usuario.

3. **GET `/api/accounts/{accountId}`**  
   Endpoint para edición, devuelve `AccountUpdateData` con datos de cuenta y cliente (balance, limites, nombre, dirección, SSN, FICO, etc.).

4. **PUT `/api/accounts/{accountId}`**  
   Recibe `AccountUpdateSubmission` (los mismos campos que `AccountUpdateData`), responde con `AccountUpdateResponse` (`success`, `data`, `errors`). El frontend vuelve a cargar los datos al recibir `success`.

---

## 📊 Modelos de Datos (resumen)

### `AccountViewResponse`
- `accountStatus`: 'Y'/'N'
- `currentBalance`, `creditLimit`, `cashCreditLimit`, `currentCycleCredit`, `currentCycleDebit`
- `customer` (ID, nombres, SSN enmascarado, FICO 300-850, fecha de nacimiento, dirección, phone numbers)
- `cardNumber`, `groupId`, `eftAccountId`, `primaryCardHolderFlag`, `programName`
- Mensajes: `errorMessage`, `infoMessage`, booleans de integridad (`inputValid`, `foundAccountInMaster`)

### `AccountUpdateData`
- Incluye campos financieros (balance, límites, fechas), personal (`firstName`, `lastName`, `ssn`, `ficoScore`, `governmentIssuedId`, `zipCode`, `stateCode`, `countryCode`) y de contacto (`phoneNumber1/2`, `addressLine*`, `eftAccountId`).
- `activeStatus` controlado por dropdown Y/N, `primaryCardIndicator` y `groupId`.

### `AccountUpdateResponse`
- `success`: boolean.
- `data`: `AccountUpdateData` actualizada.
- `errors`: array de cadenas con validaciones fallidas.

---

## 📋 Reglas de Negocio

1. `accountId` debe ser un número de 11 dígitos y no puede ser `00000000000`.
2. Solo se muestran cuentas con `status = 'Y'` como activas para acciones críticas.
3. `creditLimit` y `cashCreditLimit` siempre deben ser positivos; el `availableCredit` se calcula como `creditLimit - currentBalance`.
4. `activeStatus` solo permite valores `Y` o `N`; `zipCode` se valida con `/^\d{5}(-\d{4})?$/`.
5. `ficoScore` debe estar en el rango 300-850 y se representa con chips de color (verde ≥750, amarillo 650-749, rojo <650).
6. SSN y número de tarjeta se muestran enmascarados por defecto (`***-**-XXXX` y `****-****-****-XXXX`) hasta que el usuario habilita la visualización.
7. Actualizaciones son atómicas: si `PUT /accounts/{accountId}` devuelve errores, no se persisten cambios y se retroalimenta la UI.

---

## 🎯 Ejemplos de User Stories

1. “Como agente back-office, quiero buscar una cuenta por su ID de 11 dígitos para revisar saldos, límites y datos de contacto antes de hablar con el cliente.”
2. “Como administrador, quiero activar el modo de edición, ajustar `creditLimit` y confirmar la actualización con el diálogo de `Confirm Update` para reflejar una mejora de perfil crediticio.”
3. “Como oficial de cumplimiento, quiero que los campos de SSN y número de tarjeta estén enmascarados y solo visibles con consentimiento explícito para cumplir con PCI-DSS.”
4. “Como equipo de Quality, quiero que `F12` resetee el formulario y `F5` dispare el guardado cuando hay cambios detectados para acelerar el testing de QA.”

---

## ⚡ Factores de Aceleración

- `useAccountView` y `useAccountUpdate` encapsulan toda la lógica de API (GET/PUT), estados de carga y errores, habilitando nuevas pantallas en 1-2 días.
- `SystemHeader`, `Material-UI Grid/Card/Chip`, `LoadingSpinner` y `Alert` ya están estilizados para este módulo; basta con importar `AccountViewScreen` o `AccountUpdateScreen` y enganchar los props.
- `useMutation` con manejo de `ApiResponse` y errores reales reduce el esfuerzo de reconciliar MSW vs backend productivo.
- Hooks `updateLocalData`, `resetForm` y la comparación JSON (`originalData`) tiran de un patrón reutilizable para detectar “dirty state” en otros formularios.

---

## 📋 Dependencias

- **Librerías**: `@mui/material`, `@mui/icons-material`, `react-router-dom`, `@reduxjs/toolkit` (para layout general), `msw` (mocks de account).
- **Servicios internos**: `apiClient`, `useMutation` de `app/hooks/useApi.ts`, `SystemHeader`, `LoadingSpinner`.
- **Datos de sesión**: lee `localStorage.userRole` para autorizar rutas y decidir la navegación entre `/menu/admin` y `/menu/main`.
- **Mocks**: `app/mocks/accountHandlers.ts` provee respuestas para `/account-view`, `/account-view/initialize`, `/accounts/{id}` y escenarios de error.

---

## 🧪 Testing y Mocking

1. `app/mocks/accountHandlers.ts` registra handlers para:
   - `GET /api/account-view` con query params (padded).
   - `GET /api/account-view/initialize`.
   - `GET /api/accounts/:accountId`.
   - `PUT /api/accounts/:accountId`.
   - `GET /api/account-view/test-accounts` y `/test-error/:errorType` para QA.
2. Los tests de integración insertan test accounts (`11111111111` … `44444444444`) y validan show/hide de datos sensibles.
3. GitHub Actions (si aplica) se apegan a la política de 95%+ de cobertura de documentación (referencia en `docs/README.md`).

---

## ⚡ Presupuestos de Performance

- Consultas de vista (`/account-view`) deben responder en < 500ms (P95) debido a la experiencia de back-office que necesita datos instantáneos.
- Actualizaciones (`PUT /accounts/{id}`) en < 1s (P95) y sin bloqueos de UI (spinner y diálogo de confirmación).
- El hook `useAccountUpdate` mantiene solo 3 operaciones simultáneas (GET, PUT, comparación) para no saturar el thread de React.

---

## 🚨 Riesgos y Tech Debt

1. Falta de i18n (todo en inglés) expone riesgo para usuarios de habla hispana; implementar `react-i18next` antes de nuevas funcionalidades críticas.
2. Validaciones del frontend están parcialmente comentadas (ver líneas 87-104 en `AccountUpdateScreen.tsx`); hay que limpiar antes de habilitar producción.
3. No hay auditoría de cambios (no se registra quién modificó qué). Se recomienda Spring Data Envers o un audit trail en la capa de servicio.
4. No existe bloqueo optimista (`@Version` en entidades), por lo que varias actualizaciones concurrentes pueden pisarse.

---

## 📈 Métricas de Éxito

- **Funcional**: 100% de búsquedas exitosas por ID válidos, 0 errores críticos en validaciones, navigation intacta para admins y back-office.
- **Técnica**: Login + carga de cuenta < 500ms; `PUT` responde en < 1s; `hasChanges` detecta cambios en menos de 10ms.
- **Negocio**: Reducción de 30% en tickets de soporte por datos desactualizados; tasa de aprobación de cambios > 95%.

---

## 🔄 Flujo de Usuario

1. El operador accede a `/account-view`, el hook `initializeScreen` carga metadata, y `SystemHeader` muestra `transactionId = CAVW`.
2. Se busca el número de cuenta (11 dígitos), se muestra información en 4 tarjetas y se pueden alternar los datos sensibles.
3. Para modificar, se navega a `/account-update`, se activa `Edit Mode`, se ajustan campos y se guarda con `F5` o botón “Update Account”.
4. El hook `useAccountUpdate` llama a `PUT /accounts/{id}` y refresca el estado si `success`.
5. El operador vuelve al menú gracias a `handleExit` (F3/Escape) que revisa `userRole`.

---

## 📚 Referencias

- `app/components/account/AccountViewScreen.tsx`
- `app/components/account/AccountUpdateScreen.tsx`
- `app/pages/AccountViewPage.tsx`
- `app/pages/AccountUpdatePage.tsx`
- `app/hooks/useAccountView.ts`
- `app/hooks/useAccountUpdate.ts`
- `app/hooks/useApi.ts`
- `app/mocks/accountHandlers.ts`
