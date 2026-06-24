# AGENTS.md

## Resumen de arquitectura

- Proyecto Android/Kotlin multi-módulo llamado `KeyMapperFoss`. Los módulos Gradle declarados son `app`, `base`, `api`, `system`, `common`, `data`, `sysbridge`, `evdev` y `systemstubs`.
- `app`: aplicación FOSS, wiring de Hilt, navegación principal, `MainFragment`, `AndroidManifest` y ViewModels específicos de la variante FOSS.
- `base`: lógica de dominio/UI compartida: key maps, triggers, actions, constraints, detection, floating button models, pantallas Compose y controladores de accessibility service base.
- `data`: persistencia Room/DataStore, entidades, DAOs, repositorios, migraciones y type converters.
- `system`: adaptadores Android concretos para capacidades del sistema.
- `api`: interfaces AIDL/IPC usadas por servicios externos o bridges.
- `sysbridge` y `evdev`: integración nativa/sistema para funciones avanzadas de input; pueden requerir NDK/Rust según la tarea.
- `common`: utilidades comunes sin dependencia fuerte de UI.
- `systemstubs`: stubs/compatibilidad para APIs del sistema.

## Comandos de build/test/lint

- Listar tareas disponibles: `./gradlew tasks --all`.
- Build general: `./gradlew build`.
- Ensamblar app debug: `./gradlew :app:assembleDebug`.
- Compilar fuentes debug sin instalar: `./gradlew :app:compileDebugSources`.
- Unit tests por módulo: `./gradlew :base:testDebugUnitTest`, `./gradlew :data:testDebugUnitTest`, `./gradlew :app:testDebugUnitTest`.
- Android lint: `./gradlew :app:lintDebug` o por módulo `:<module>:lintDebug`.
- Ktlint: el plugin `org.jlleitschuh.gradle.ktlint` está aplicado en módulos Kotlin principales; usar `./gradlew ktlintCheck` para verificación y `./gradlew ktlintFormat` solo cuando se quiera formatear código.
- Tests instrumentados: `./gradlew connectedDebugAndroidTest` requiere dispositivo/emulador Android.
- Nota de entorno: `local.properties` existe localmente y no debe committearse. En esta copia Gradle puede advertir que `ndk.dir` está deprecado; no corregirlo salvo que la tarea lo pida explícitamente.

## Reglas de estilo y trabajo

- Seguir Kotlin style + ktlint existente; cambios pequeños y quirúrgicos.
- No envolver imports en `try/catch`.
- No introducir dependencias nuevas salvo necesidad clara.
- No modificar `local.properties`, secretos, endpoints privados ni configuración de firma.
- Respetar compatibilidad de backups/JSON: muchas entidades tienen constantes `NAME_*` con comentarios `DON'T CHANGE THESE`; no renombrarlas ni cambiar IDs/tipos serializados sin migración y compatibilidad.
- Si se cambia Room schema, incrementar `DATABASE_VERSION`, añadir migración/auto-migración y revisar backups.
- Evitar bloquear el hilo principal; las capas existentes usan coroutines, Flow/StateFlow, Room y repositorios.
- En Android, extremar cuidado con AccessibilityService, overlays, permisos, foreground/background restrictions, MediaProjection, targetSdk, manifest y notificaciones.

## Zonas relevantes para floating buttons

- Modelos de dominio de floating buttons:
  - `base/src/main/java/io/github/sds100/keymapper/base/floating/FloatingButtonData.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/floating/FloatingLayoutData.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/floating/FloatingButtonAppearance.kt`
- Trigger key de floating button y mapeo con entidad:
  - `base/src/main/java/io/github/sds100/keymapper/base/trigger/FloatingButtonKey.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/FloatingButtonKeyEntity.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/TriggerKeyEntity.kt`
- UI/configuración de triggers:
  - `base/src/main/java/io/github/sds100/keymapper/base/trigger/BaseTriggerScreen.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/trigger/BaseConfigTriggerViewModel.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/trigger/TriggerDiscoverScreen.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/trigger/TriggerKeyOptionsBottomSheet.kt`
  - `app/src/main/java/io/github/sds100/keymapper/trigger/ConfigTriggerViewModel.kt`
- Detección/ejecución:
  - `base/src/main/java/io/github/sds100/keymapper/base/detection/KeyMapAlgorithm.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/detection/KeyMapDetectionController.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/detection/DetectKeyMapsUseCase.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/system/accessibility/BaseAccessibilityServiceController.kt`
- Persistencia Room/repositories:
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/FloatingButtonEntity.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/FloatingLayoutEntity.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/FloatingButtonEntityWithLayout.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/entities/FloatingLayoutEntityWithButtons.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/db/dao/FloatingButtonDao.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/db/dao/FloatingLayoutDao.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/repositories/FloatingButtonRepository.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/repositories/FloatingLayoutRepository.kt`
  - `data/src/main/java/io/github/sds100/keymapper/data/db/AppDatabase.kt`
- Backups/import/export:
  - `base/src/main/java/io/github/sds100/keymapper/base/backup/BackupManager.kt`
  - `base/src/main/java/io/github/sds100/keymapper/base/backup/BackupContent.kt`
- Overlay/renderizado: en esta variante FOSS no aparece una implementación completa de editor/render de overlay en el árbol inspeccionado. La integración visible es el `BaseAccessibilityServiceController`, que solicita `FLAG_RETRIEVE_INTERACTIVE_WINDOWS` para detectar cuándo mostrar/ocultar overlays, y `KeyMapDetectionController.onFloatingButtonDown/Up`, que envía eventos al algoritmo. La implementación visual concreta puede estar en otra variante/módulo no incluido o pendiente.

## Triggers, actions, constraints y modelo de datos

- Triggers: dominio/UI en `base/src/main/java/io/github/sds100/keymapper/base/trigger`; serialización en `data/entities/TriggerEntity.kt` y `data/entities/TriggerKeyEntity.kt`; detección en `base/detection/KeyMapAlgorithm.kt`.
- Actions: dominio/UI/ejecución en `base/src/main/java/io/github/sds100/keymapper/base/actions`; serialización en `data/entities/ActionEntity.kt`; la ejecución se conecta desde `PerformActionsUseCaseImpl`.
- Constraints: dominio/UI/detección en `base/src/main/java/io/github/sds100/keymapper/base/constraints`; serialización en `data/entities/ConstraintEntity.kt`; evaluación vía `DetectConstraintsUseCaseImpl`.
- Key maps: `data/entities/KeyMapEntity.kt` guarda `trigger`, `actionList`, `constraintList`, `constraintMode`, flags, enabled, uid y group uid; `base/keymaps/KeyMap.kt` y mappers convierten entre entidades y dominio.
- Room: `data/db/AppDatabase.kt` registra entidades, versión y migraciones; DAOs en `data/db/dao`; repositorios en `data/repositories`.

## Advertencias para futuros agentes

- No asumir que la implementación visual de floating overlay está presente en FOSS; verificar si existe una variante proprietary/pro o ramas no incluidas antes de diseñar APIs definitivas.
- No tocar IDs serializados, nombres JSON ni constantes de entidades sin plan de migración y compatibilidad con backups.
- `FloatingButtonData.Location` depende de orientación y tamaño de pantalla; un editor de cuadrícula debe preservar o migrar cuidadosamente esa semántica.
- Floating buttons también son trigger keys; cualquier cambio en layouts/buttons puede dejar triggers apuntando a botones borrados. Mantener estados de error existentes para botones eliminados.
- Revisar compra/feature gating: en FOSS, `ConfigTriggerViewModel` redirige atajos de floating buttons/assistant a pantalla de soporte/compra.
- Los cambios de overlay/accessibility deben probarse en dispositivo real con rotación, lock screen, IME visible, status bar y apps con ventanas múltiples.
- No ejecutar formateadores globales si solo se cambia documentación o una zona pequeña.

## Propuesta de enfoque para editor visual de cuadrícula

1. Confirmar dónde vive la implementación visual real de floating overlays. Si no está en FOSS, diseñar primero interfaces/modelos compartidos en `base`/`data` y dejar la implementación concreta para el módulo/variante que renderiza overlays.
2. Reutilizar `FloatingLayoutData` como agregado principal y `FloatingButtonData` como item. Añadir un modelo de editor separado si hace falta representar grid temporal, selección, snap, preview o cambios sin guardar.
3. Evitar cambiar la semántica de `Location` al inicio: implementar snap-to-grid calculando `x/y` finales en coordenadas existentes y conservando `orientation`/`displaySize`.
4. Añadir casos de uso/repositorio mínimos para cargar un layout con buttons, modificar posiciones en lote y guardar de forma transaccional si se edita más de un botón.
5. Integrar navegación desde `onEditFloatingButtonClick`/`onEditFloatingLayoutClick` siguiendo patrones de `BaseConfigTriggerViewModel` y pantallas Compose existentes.
6. Crear UI Compose del editor con preview de botones, guías de cuadrícula, drag handles, snap configurable y acciones guardar/cancelar; evitar que la pantalla escriba directamente a Room.
7. Validar con tests de mappers/casos de uso para posiciones, orientación y botones eliminados; pruebas manuales en dispositivo para overlay real, rotación, IME/status bar y lock screen.
