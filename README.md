# 🚀 WhatsApp Autopilot IA: Tu Asistente Personal con Memoria (Versión Robusta)

**Transforma tu WhatsApp en un asistente inteligente que responde mensajes 24/7, recuerda conversaciones, sigue tus instrucciones y es estable ante fallos y mensajes simultáneos.**

Este proyecto es una guía completa y probada para construir un sistema de auto-respuesta para WhatsApp (o cualquier app de mensajería) en Android. A diferencia de los respondedores básicos, este asistente se conecta a una API de IA (compatible con OpenAI GPT, Google Gemini, etc.) para mantener conversaciones coherentes y personalizadas, gracias a un sistema de memoria persistente y múltiples capas de seguridad.

---

## ✨ Características Principales

* **Memoria Persistente:** Recuerda el historial de cada conversación de forma individual.
* **Personalización Dinámica:** Saluda a cada contacto por su primer nombre y puede adaptar sus instrucciones sobre la marcha.
* **Filtrado Inteligente Avanzado:** Ignora automáticamente notificaciones resumen, chats de grupo y sus propias respuestas para evitar bucles infinitos.
* **Gestión de Colisiones Profesional:** Utiliza un sistema de "cerrojo" (mutex) y un "watchdog" de auto-reseteo para manejar múltiples mensajes simultáneos de forma ordenada, sin cruzar conversaciones ni bloquearse.
* **Sistema a Prueba de Fallos:** Diseñado para ser robusto, con acciones que continúan tras un error para garantizar que el sistema siempre se recupere y libere los bloqueos.
* **Base de Conocimiento Centralizada:** Se programa a través de un `prompt` en formato JSON, permitiendo definir fácilmente reglas, precios y la personalidad del asistente.

## 📋 Requisitos Previos

1.  **Tasker:** La última versión instalada desde la Play Store.
2.  **Plugin AutoNotification:** Instalado desde la Play Store.
3.  **API Key de un Proveedor de IA:** Una clave de API válida de [OpenAI](https://platform.openai.com/api-keys), [Google AI Studio](https://aistudio.google.com/app/apikey), o similar.
4.  **Permisos Críticos en Android:** Para la aplicación **Tasker**, es **esencial** haber concedido los siguientes permisos desde los Ajustes de Android:
    * `Acceso a Notificaciones` (para el plugin AutoNotification).
    * `Batería` → **"Sin restricciones"**. Vital para evitar que el sistema operativo detenga la tarea.
    * `Permisos` → `Archivos y contenido multimedia` → **"Permitir la gestión de todos los archivos"**.

---

## 🛠️ Guía de Implementación Paso a Paso

### Paso 1: Preparación del Entorno

#### 1.1 Guardar la Clave de API
* **Por qué:** Guardamos la clave en una variable global para no tener que escribirla repetidamente y para mantenerla segura en un solo lugar.
* **Acción:**
    1.  Abre Tasker, ve a la pestaña **VARS**.
    2.  Crea una nueva variable global llamada `%AI_API_KEY`.
    3.  Pega tu clave secreta de la API en el campo de valor.

#### 1.2 Crear Carpeta de Historial
* **Por qué:** Necesitamos un lugar dedicado para almacenar los archivos `.json` que contendrán el historial de cada conversación.
* **Acción:**
    1.  Usando un explorador de archivos, ve a tu almacenamiento interno.
    2.  Dentro de la carpeta `Tasker`, crea una nueva carpeta llamada `ChatHistory`.

### Paso 2: Creación del Perfil (El Activador Inteligente)

* **Por qué:** Este perfil es el "guardia de seguridad". Su configuración es crucial para asegurar que la tarea solo se active con notificaciones que nos interesan.
* **Nombre Sugerido para el Perfil:** `Asistente IA Trigger`
* **Acción:**
    1.  Ve a la pestaña **PERFILES** y crea un nuevo perfil de **Evento**.
    2.  Selecciona `Plugin` → `AutoNotification` → `Intercept`.
    3.  Configura el perfil con las siguientes reglas:
        * **Apps:** Selecciona `WhatsApp` y/o `WhatsApp Business`.
        * **Has Reply Action:** Marca esta casilla (Activado).
            > *Asegura que es un mensaje directo al que se puede responder.*
        * **Ignore Group Summaries:** Marca esta casilla (Activado).
            > *Descarta notificaciones resumen como "312 mensajes nuevos".*
        * **Title Filter:** Escribe `.*:.*`, marca la casilla **`Regex`** y marca la casilla **`Invert`**.
            > *Ignora cualquier mensaje de un chat de grupo.*
    4.  **Mantén pulsado el nombre del perfil** y toca el **icono de engranaje ⚙️** (Propiedades).
    5.  Busca **`Cooldown Time`** y establécelo en **20 segundos**.
        > *¡Crucial! Evita bucles y que WhatsApp/Android bloqueen las respuestas por ser demasiado rápidas, haciendo que el bot parezca más "humano".*
    6.  Al salir, asocia este perfil a una **Tarea Nueva** y nómbrala `Asistente IA WhatsApp`.

### Paso 3: Construcción de la Tarea Principal (El Cerebro)

Esta es la secuencia final y optimizada de acciones para la tarea `Asistente IA WhatsApp`.

#### Fase 1: Failsafe, Cerrojo y Recopilación de Datos
* **1. Failsafe del Cerrojo (Watchdog)**
    * `Tarea` → `Si`. Condición: `%AutopilotIsRunning` `~` `1` **Y** `%TIMES - %AutopilotStartTime` `>` `120`.
    * **Por qué:** Es un sistema de auto-reparación. Si el cerrojo se ha quedado "pegado" por más de 2 minutos (120s), esta acción lo libera forzosamente.

* **2. Liberar Cerrojo Atascado**
    * `Variables` → `Borrar Variable`. Nombre: `%AutopilotIsRunning`.
    * *Esta acción está anidada dentro del `Si` anterior.*

* **3. Fin Si**
    * `Tarea` → `Fin Si`.

* **4. Detener si está Ocupado (Cerrojo)**
    * `Tarea` → `Detener`. Con Condición `Si`: `%AutopilotIsRunning` `~` `1`.
    * **Por qué:** Es el semáforo principal. Si la tarea ya está procesando un mensaje, cualquier nueva activación se detiene aquí.

* **5. Activar Cerrojo**
    * `Variables` → `Definir Variable`. Nombre: `%AutopilotIsRunning`, A: `1`.
    * **Por qué:** "Cierra la puerta" para indicar que el proceso ha comenzado.

* **6. Registrar Hora de Inicio del Cerrojo**
    * `Variables` → `Definir Variable`. Nombre: `%AutopilotStartTime`, A: `%TIMES`.
    * **Por qué:** Guarda la hora actual. Es necesario para que el Failsafe del paso 1 pueda calcular si el cerrojo se ha quedado atascado.

* **7. Forzar Lectura de la Notificación**
    * `Plugin` → `AutoNotification` → `Query`. Configuración: En `Apps`, selecciona tus apps de WhatsApp. En `Title`, pon `%antitle`. En `Text`, pon `%antext`.
    * **Por qué:** Garantiza que la tarea obtenga una copia fresca y estable de las variables de la notificación, evitando "variables fantasma".

* **8. Detener si es un Mensaje Propio (Anti-Bucle)**
    * `Tarea` → `Detener`. Con Condición `Si`: `%antitle(1)` `~R` `(?i)You|Tú`.
    * **Por qué:** Revisa si el remitente es "You" o "Tú". Si es así, detiene la tarea para evitar que el asistente se responda a sí mismo en un bucle infinito.

* **9. Copiar Nombre Completo**
    * `Variables` → `Definir Variable`. Nombre: `%nombre_para_split`, A: `%antitle(1)`.

* **10. Extraer Primer Nombre**
    * `Variables` → `Variable Split`. Nombre: `%nombre_para_split`, Divisor: ` ` (un espacio).
    * **Por qué:** El primer nombre queda en `%nombre_para_split1` para un saludo más personal.

#### Fase 2: Configuración y Gestión de Memoria
* **11-15. Variables y Prompt Dinámico**
    * **11. `Definir Variable`**: `%timeout_sesion_segundos`, A: `900`.
    * **12. `Definir Variable`**: `%ruta_historiales`, A: `Tasker/ChatHistory`.
    * **13. `Definir Variable`**: `%prompt_sistema`, A: *Pega aquí tu prompt en formato JSON*.
    * **14. `Definir Variable`**: `%prompt_dinamico`, A: `%prompt_sistema`.
    * **15. `Búsqueda Reemplazar Variable`**: Variable: `%prompt_dinamico`, Búsqueda: `%%USERNAME%%`, Reemplazar Con: `%nombre_para_split1`.

* **16. Limpiar Nombre para Archivo**
    * `Variables` → `Búsqueda Reemplazar Variable`. Variable: `%antitle(1)`, Búsqueda: `\W`, Marcar `Regex`, Reemplazar Con: `_`.

* **17. Construir Ruta del Archivo**
    * `Variables` → `Definir Variable`. Nombre: `%archivo_historial`, A: `%ruta_historiales/%antitle(1)_history.json`.

* **18-26. Lógica de Timeout y Lectura del Historial**
    * **18. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Existe`, Guardar En: `%historial_existe`.
    * **19. `Definir Variable`**: `%historial_json_raw`, A: `[]`.
    * **20. `Si`**: `%historial_existe` `~` `true`.
        * **21. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Fecha de Modificación`, Guardar En: `%fecha_ultimo_mensaje`.
        * **22. `Definir Variable`**: `%segundos_desde_ultimo`, A: `%TIMES - %fecha_ultimo_mensaje`, Marcar `Hacer Cuentas`.
        * **23. `Si`**: `%segundos_desde_ultimo > %timeout_sesion_segundos`.
            * **24. `Borrar Fichero`**: Ruta: `%archivo_historial`, Marcar `Continuar Tarea Tras Error`.
        * **25. `Si no`** (Else).
            * **26. `Leer Fichero`**: Fichero: `%archivo_historial`, A Variable: `%historial_json_raw`.
        * **27. `Fin Si`**.
    * **28. `Fin Si`**.
    * **Por qué:** Esta estructura robusta comprueba si un archivo existe y es reciente. Si es viejo, lo borra; si es reciente, lo lee. Si no existe, no hace nada. Esto evita cualquier error de "archivo no encontrado".

#### Fase 3: Comunicación con la IA
* **29. Preparar Envío (JavaScriptlet)**
    * `Código` → `JavaScriptlet`. **Código:**
        ```javascript
        const oldHistoryJson = historial_json_raw;
        const systemPrompt = prompt_dinamico;
        const userMessage = antext;
        let messages = [];
        if (oldHistoryJson === '[]' || JSON.parse(oldHistoryJson).length === 0) {
            messages.push({ role: "system", content: systemPrompt });
        } else {
            messages = JSON.parse(oldHistoryJson);
        }
        messages.push({ role: "user", content: userMessage });
        setLocal('messages_array_json', JSON.stringify(messages));
        ```

* **30. Petición HTTP**
    * `Red` → `Petición HTTP`.
    * **Configuración:** Método `POST`, URL de la API, Cabeceras con `%AI_API_KEY`, Cuerpo con `%messages_array_json`, Timeout `60s`, Marcar `Confía en cualquier certificado` y **`Continuar Tarea Tras Error`**.

* **31. Procesar Respuesta (JavaScriptlet)**
    * `Código` → `JavaScriptlet`. **Código:**
        ```javascript
        try {
            const data = JSON.parse(http_data);
            if (data.error) {
                setLocal('gpt_reply', 'Error API: ' + data.error.message);
                setLocal('new_history_to_save', historial_json_raw);
            } else {
                const replyText = data.choices[0].message.content.trim();
                setLocal('gpt_reply', replyText);
                let messages = JSON.parse(messages_array_json);
                messages.push({ role: "assistant", content: replyText });
                const newHistoryToSave = JSON.stringify(messages, null, 2);
                setLocal('new_history_to_save', newHistoryToSave);
            }
        } catch (error) {
            setLocal('gpt_reply', 'Error procesando respuesta.');
            setLocal('new_history_to_save', historial_json_raw);
        }
        ```

#### Fase 4: Guardado, Respuesta y Limpieza
* **32. Guardar Historial Actualizado**
    * `Fichero` → `Escribir Fichero`. Configuración: Fichero: `%archivo_historial`, Texto: `%new_history_to_save`.

* **33. Pausa de "Humanización"**
    * `Tarea` → `Esperar`. Segundos: `3`.
    * **Por qué:** Añade un retardo antes de responder para que el asistente parezca más humano y evitar que WhatsApp lo marque como spam.

* **34. Responder Mensaje**
    * `Plugin` → `AutoNotification` → `Reply`.
    * **Configuración:** Reply Text: `%gpt_reply`, Reply Action ID: `%anreplyaction(1)`, **DESMARCAR** `Cancel Before Replying` y Marcar **`Continuar Tarea Tras Error`**.

* **35. Liberar Cerrojo (Limpieza Final)**
    * `Variables` → `Borrar Variable`. Nombre: `%AutopilotIsRunning`.

* **Propiedades de Tarea:**
    * Dentro de la tarea, toca el **icono de engranaje ⚙️**.
    * En **Manejo de Colisiones**, selecciona `Abortar Tarea Nueva`.

### ✍️ Paso 4: Ejemplo de Prompt para el Asistente

Puedes usar esta plantilla como base para tu variable `%prompt_sistema`.

```json
{
  "asistente": "Autopilot",
  "propósito": "Soy un asistente virtual diseñado para apoyar, informar y acompañar a los usuarios. Mi objetivo es ofrecer respuestas útiles, claras y atractivas, con una personalidad cálida, profesional y adaptable.",
  "personalidad": [
    "Carismática, empática y fácil de conversar.",
    "Profesional cuando el contexto lo requiere, pero también relajada y divertida.",
    "Curiosa, creativa y proactiva."
  ],
  "reglasDeComportamiento": {
    "saludoInicial": "Tu primera respuesta SIEMPRE debe empezar saludando al usuario por su nombre de pila. Su nombre es '%%USERNAME%%'.",
    "reglaDeOro": "Nunca te presentes dos veces en la misma conversación."
  },
  "capacidades": [
    "Responder preguntas con precisión y contexto.",
    "Redactar textos.",
    "Resumir, analizar y organizar información."
  ],
  "límites": [
    "No comparto información privada o sensible.",
    "No doy opiniones personales sobre temas delicados.",
    "No simulo emociones humanas reales."
  ]
}
```

## 🚀 Prueba Final
Para probar el sistema, **no ejecutes la tarea manualmente**. La única forma correcta es:
1. Asegúrate de que el perfil y la tarea estén activados.
2. Envíate un mensaje real desde otro número a la app de WhatsApp que has configurado.
¡Felicidades por completar este increíblemente avanzado y robusto proyecto de Tasker!
