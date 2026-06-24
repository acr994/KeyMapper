Informe técnico: sistema de floating buttons
Alcance y estado del repositorio
No modifiqué código ni documentación. Leí AGENTS.md primero, como indica la instrucción del repositorio, y limité el análisis al sistema de floating buttons y sus puntos de integración.

Comandos usados para el análisis:

cat AGENTS.md

rg -n "FloatingButton|floating button|floatingButtons|Overlay|overlay|WindowManager|addView|updateViewLayout|TYPE_ACCESSIBILITY_OVERLAY" base app system data -g '*.kt'

rg -n "FloatingButtonAppearance|backgroundOpacity|borderOpacity|buttonSize|FloatingButtonData\\(" base app data system -g '*.kt'

rg -n "LONG_PRESS|DOUBLE_PRESS|longPress|doublePress|setClickType|ClickType" base/src/main/java/io/github/sds100/keymapper/base/detection/KeyMapAlgorithm.kt base/src/main/java/io/github/sds100/keymapper/base/trigger/TriggerMode.kt base/src/main/java/io/github/sds100/keymapper/base/keymaps/ClickType.kt

Varios nl -ba <archivo> | sed -n '<rango>p' para obtener líneas citables.

1. Clases implicadas
Modelos de dominio de floating buttons
El núcleo de dominio está en base/src/main/java/io/github/sds100/keymapper/base/floating.

FloatingButtonData representa un botón flotante completo: uid, layoutUid, layoutName, apariencia, localización, flags de visibilidad sobre status bar/IME y bloqueo de posición. 

FloatingButtonData.Location guarda x, y, orientación original y tamaño de pantalla, lo que permite recolocar el overlay de forma consistente entre orientaciones. 

FloatingButtonAppearance define texto, tamaño, opacidad de borde/fondo, límites de tamaño y detección de botón invisible. 

FloatingLayoutData representa un layout con uid, name y lista de FloatingButtonData. 

Mappers dominio ↔ Room
FloatingButtonEntityMapper convierte entre FloatingButtonEntity y FloatingButtonData, incluyendo apariencia, localización, flags de status bar/IME y posición bloqueada. 

FloatingLayoutEntityMapper convierte FloatingLayoutEntityWithButtons a FloatingLayoutData y FloatingLayoutData a FloatingLayoutEntity. 

Trigger key específico
FloatingButtonKey es el trigger key que enlaza un key map con un botón flotante mediante buttonUid, guarda opcionalmente el FloatingButtonData resuelto y soporta clickType. 

FloatingButtonKey declara explícitamente que permite pulsación larga y doble pulsación. 

Su mapper convierte FloatingButtonKeyEntity a dominio resolviendo SHORT_PRESS, LONG_PRESS y DOUBLE_PRESS. 

La conversión inversa persiste el clickType como constantes de TriggerKeyEntity. 

Configuración de triggers
ConfigTriggerUseCaseImpl inyecta FloatingButtonRepository y FloatingLayoutRepository, expone floatingButtonToUse y ofrece getFloatingLayoutCount() y addFloatingButtonTriggerKey(buttonUid). 

ConfigTriggerDelegate.addFloatingButtonTriggerKey() crea el FloatingButtonKey; si el trigger está en modo paralelo hereda el clickType del modo, y si está en secuencia o indefinido usa SHORT_PRESS. 

ConfigKeyMapStateImpl observa floatingButtonRepository.buttonsList y actualiza los datos embebidos de los FloatingButtonKey cuando cambian los botones. 

Detección/algoritmo
KeyMapDetectionController expone onFloatingButtonDown(button) y onFloatingButtonUp(button), reenviando ambos eventos al algoritmo. 

KeyMapAlgorithm crea FloatingButtonEvent(buttonUid, clickType = null) en down/up y lo procesa como cualquier otro evento de trigger. 

El algoritmo compara eventos de floating button por buttonUid y clickType. 

2. Modelos de datos
Entidades Room principales
FloatingLayoutEntity
FloatingLayoutEntity se guarda en la tabla floating_layouts, con uid como primary key y name único. 

Sus nombres serializados uid y name están marcados como compatibilidad JSON/backup. 

FloatingButtonEntity
FloatingButtonEntity se guarda en la tabla floating_buttons, con foreign key a FloatingLayoutEntity; al borrar un layout se borran sus botones por onDelete = CASCADE. 

Campos persistidos:

Identidad y asociación: uid, layoutUid. 

Apariencia: text, buttonSize, borderOpacity, backgroundOpacity. 

Posición: x, y, orientation, displayWidth, displayHeight. 

Opciones de overlay: showOverStatusBar, showOverInputMethod, isPositionLocked. 

La entidad mantiene constantes NAME_* para serialización/backup y un DESERIALIZER que reconstruye todos los campos persistidos. 

Relaciones Room
FloatingButtonEntityWithLayout embebe un botón y relaciona layout_uid con el uid del layout. 

FloatingLayoutEntityWithButtons embebe un layout y resuelve todos los botones cuyo layout_uid apunta al layout. 

FloatingButtonKeyEntity
FloatingButtonKeyEntity es la representación serializable del trigger key; guarda buttonUid, clickType y uid. 

Su NAME_BUTTON_UID está marcado como constante de serialización que no debe cambiarse. 

3. Pantallas de configuración
Descubrimiento de triggers
La pantalla de discovery muestra una sección de floating buttons cuando showFloatingButtons es true. Esa sección contiene accesos para FLOATING_BUTTON_CUSTOM y FLOATING_BUTTON_LOCK_SCREEN. 

En la variante FOSS, TriggerScreen activa showFloatingButtons = true, pero los shortcuts avanzados quedan gateados más adelante por el ViewModel FOSS. 

Opciones del trigger key
BaseTriggerScreen conecta los callbacks onEditFloatingButtonClick y onEditFloatingLayoutClick al bottom sheet de opciones de trigger. 

TriggerKeyOptionsBottomSheet muestra el botón “configure button” solo si el estado es TriggerKeyOptionsState.FloatingButton y isPurchased es true. 

BaseConfigTriggerViewModel construye TriggerKeyOptionsState.FloatingButton con clickType, showClickTypes y el resultado de displayKeyMap.isFloatingButtonsPurchased(). 

Estado visual de items de trigger
Cuando un FloatingButtonKey tiene botón asociado, se muestra como TriggerKeyListItemModel.FloatingButton con nombre de botón, nombre de layout, clickType, tipo de enlace y posible error. 

Si el botón ya no existe, el modelo pasa a FloatingButtonDeleted, cuyo error fijo es TriggerError.FLOATING_BUTTON_DELETED. 

Implementación FOSS actual
En app/src/main/java/.../ConfigTriggerViewModel.kt, los callbacks onEditFloatingButtonClick() y onEditFloatingLayoutClick() están vacíos. 

Además, los shortcuts ASSISTANT, FLOATING_BUTTON_CUSTOM y FLOATING_BUTTON_LOCK_SCREEN navegan a una pantalla de soporte/compra en lugar de abrir configuración real. 

4. Servicios de overlay
En el árbol FOSS inspeccionado no encontré una clase concreta tipo FloatingButtonOverlayService, WindowManager.addView, TYPE_ACCESSIBILITY_OVERLAY o renderer visual de botones flotantes.

La integración de servicio visible está en BaseAccessibilityServiceController: al configurar flags del accessibility service, solicita FLAG_RETRIEVE_INTERACTIVE_WINDOWS porque es necesario recibir eventos de ventanas y detectar cuándo mostrar/ocultar overlays. 

La ruta de eventos esperada hacia el algoritmo sí existe: algún componente de overlay externo/no presente debería llamar a KeyMapDetectionController.onFloatingButtonDown(button) y onFloatingButtonUp(button), que reenvían al algoritmo. 

Conclusión: en esta variante FOSS están presentes los modelos, persistencia, trigger keys y entrada al algoritmo; no está presente el renderer/servicio visual completo del overlay.

5. Cómo se almacenan los layouts
Tablas y relaciones
Los layouts se almacenan en floating_layouts; los botones se almacenan en floating_buttons. Los nombres de tabla y columnas están definidos en los DAOs. 

Room registra FloatingLayoutEntity y FloatingButtonEntity como entidades de la base de datos. 

La base incluye migraciones automáticas relacionadas con floating buttons: opacidades, flags de status bar/IME y bloqueo de posición. 

DAOs
FloatingLayoutDao permite obtener layout con botones por uid, observarlo como Flow, listar todos los layouts con botones, insertar, borrar, actualizar y contar layouts. 

FloatingButtonDao permite obtener botón por uid, obtenerlo con layout, observar todos los botones con layout, insertar con REPLACE, borrar y actualizar. 

Repositorios
RoomFloatingLayoutRepository expone layouts como StateFlow<State<List<FloatingLayoutEntityWithButtons>>>, inserta/actualiza en dispatcher IO y captura SQLiteConstraintException al renombrar/actualizar layouts. 

RoomFloatingButtonRepository expone buttonsList como StateFlow<State<List<FloatingButtonEntityWithLayout>>> y encapsula insert/update/delete/get. 

Sincronización con triggers
Cuando se cargan key maps o cambian botones, ConfigKeyMapStateImpl resuelve los FloatingButtonKey con los datos actuales del botón/layout. 

Trigger.updateFloatingButtonData() pone button = null si el botón ya no existe, o reconstruye FloatingButtonData si existe. 

6. Cómo se renderizan los botones
Lo que sí existe
El modelo de render/apariencia está definido:

Texto y tamaño del botón. 

Opacidad de borde y fondo. 

Tamaño mínimo, default y máximo en dp. 

Detección de botón invisible cuando no tiene texto y ambas opacidades son 0. 

La posición de render esperada está modelada por FloatingButtonData.Location: coordenadas x/y, orientación original y tamaño de display. 

Los flags showOverStatusBar, showOverInputMethod e isPositionLocked también forman parte del modelo que usaría el renderer/overlay. 

Lo que no aparece en FOSS
No encontré código en esta variante que cree vistas de overlay o pinte botones con WindowManager. La única referencia explícita a overlays en el servicio base es la necesidad de observar ventanas para mostrar/ocultar overlays. 

Inferencia técnica: el renderer real probablemente vive en una variante no incluida, fue eliminado de FOSS, o está pendiente. La parte reusable presente es el contrato de datos y la entrada al motor de detección.

7. Cómo se gestionan doble pulsación y pulsación larga
Tipos de click
Los tipos están centralizados en ClickType: SHORT_PRESS, LONG_PRESS y DOUBLE_PRESS. 

FloatingButtonKey permite long press y double press. 

Persistencia del click type
FloatingButtonKeyEntity persiste clickType como entero. 

FloatingButtonKey.fromEntity() traduce TriggerKeyEntity.SHORT_PRESS, LONG_PRESS y DOUBLE_PRESS al enum de dominio. 

FloatingButtonKey.toEntity() hace la conversión inversa. 

Asignación inicial al configurar
Cuando se añade un floating button trigger key, si el trigger está en modo paralelo hereda el clickType de TriggerMode.Parallel; en modo secuencia o indefinido se crea como SHORT_PRESS. 

Flujo de eventos
El overlay debería generar eventos down y up con un buttonUid. El algoritmo los convierte en FloatingButtonEvent(buttonUid, clickType = null). 

Después, el algoritmo etiqueta internamente el evento como short/long/double mediante helpers withShortPress, withLongPress y withDoublePress. 

Para que un floating button coincida, el buttonUid y el clickType deben coincidir con el FloatingButtonKey. 

Long press
El algoritmo toma delays por trigger si existen, o usa defaults observados desde preferencias. 

En triggers de secuencia, si el tiempo entre down/up alcanza longPressDelay(trigger), el evento se codifica como withLongPress; si no, como short o double según corresponda. 

En triggers paralelos/single-key también se comprueba si el tiempo supera longPressDelay para marcar una pulsación larga exitosa. 

Double press
Durante la preparación de key maps, el algoritmo recoge keys con ClickType.DOUBLE_PRESS cuando el modo no es paralelo y activa detección de doble pulsación. 

En runtime, para cada candidato de doble pulsación, compara contra event.withDoublePress, incrementa estado interno y maneja dos fases: SINGLE_PRESSED inicia timeout; DOUBLE_PRESSED marca éxito y resetea estado/timeout. 

El timeout de doble pulsación usa trigger.doublePressDelay o el default global. 

8. Limitaciones y advertencias técnicas
En la variante FOSS inspeccionada no está la implementación visual del overlay ni un servicio específico que renderice botones flotantes; solo están los modelos, persistencia, UI de trigger, gating FOSS y entrada de eventos al algoritmo.

FloatingButtonData.Location mezcla coordenadas con orientación/tamaño de display; un editor de cuadrícula debe preservar esa semántica para no romper rotación ni layouts existentes. 

Borrar layouts borra botones por cascade, y los triggers pueden quedar apuntando a botones inexistentes; el estado FloatingButtonDeleted ya cubre esa situación. 

No conviene cambiar nombres JSON/Room existentes sin migración y revisión de backup, porque las entidades usan constantes marcadas para serialización estable. 

Files changed
Ninguno.

What changed
Nada; solo análisis técnico.

Tests/build/lint run and real result
No ejecuté tests/build/lint porque no hubo cambios de código ni documentación en este turno.

Risks or limitations
La principal limitación del análisis es que el renderer/servicio visual de overlay no aparece en el árbol FOSS inspeccionado; el informe describe la integración disponible y marca esa ausencia explícitamente.

Manual checks if needed
Si se continúa con el editor visual, verificar en dispositivo real: rotación, IME visible, status bar, lock screen, ventanas múltiples y que los triggers existentes con FloatingButtonKey sigan resolviendo botón/layout correctamente.




