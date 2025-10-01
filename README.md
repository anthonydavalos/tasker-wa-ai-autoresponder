# 🚀 WhatsApp Autopilot IA: Tu Asistente Personal con Memoria

**Transforma tu WhatsApp en un asistente inteligente que responde mensajes 24/7, recuerda conversaciones y sigue tus instrucciones. ¡Todo con el poder de Tasker y la IA!**

Este proyecto es una guía completa para construir un sistema de auto-respuesta para WhatsApp (o cualquier app de mensajería) en Android. A diferencia de los respondedores básicos, este asistente se conecta a una API de IA (compatible con OpenAI GPT, Google Gemini, etc.) para mantener conversaciones coherentes y personalizadas con cada contacto, gracias a un sistema de memoria persistente.

---

## ✨ Características Principales

* **Memoria Persistente:** Recuerda el historial de cada conversación de forma individual.
* **Personalización Dinámica:** Saluda a cada contacto por su nombre y puede adaptar sus instrucciones sobre la marcha.
* **Filtrado Inteligente:** Ignora automáticamente notificaciones resumen y mensajes de chats de grupo.
* **Gestión de Colisiones Profesional:** Utiliza un sistema de "cerrojo" (mutex) para manejar múltiples mensajes simultáneos de forma ordenada y sin cruzar conversaciones.
* **Sistema a Prueba de Fallos:** Diseñado para ser robusto, con mecanismos que previenen errores comunes y aseguran que el sistema siempre se recupere.
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

* **Por qué:** Este perfil es el "guardia de seguridad". Su configuración es crucial para asegurar que la tarea solo se active con notificaciones de mensajes individuales y respondibles, ignorando todo lo demás.
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
    4.  Al salir, asocia este perfil a una **Tarea Nueva** y nómbrala `Asistente IA WhatsApp`.

### Paso 3: Construcción de la Tarea Principal (El Cerebro)

Esta es la secuencia exacta y corregida de **30 acciones** que componen la tarea.

#### Fase 1: Recopilación y Preparación de Datos
* **1. Detener si está Ocupado (Cerrojo)**
    * `Tarea` → `Detener`. Con Condición `Si`: `%AutopilotIsRunning` `~` `1`.
    * **Por qué:** Evita colisiones. Si la tarea ya está procesando un mensaje, cualquier nueva activación se detiene aquí.

* **2. Activar Cerrojo**
    * `Variables` → `Definir Variable`. Nombre: `%AutopilotIsRunning`, A: `1`.
    * **Por qué:** "Cierra la puerta" para indicar que el proceso ha comenzado. Al ser una variable global, es visible para todas las demás ejecuciones.

* **3. Forzar Lectura de la Notificación**
    * `Plugin` → `AutoNotification` → `Query`. Configuración: En `Apps`, selecciona tus apps de WhatsApp.
    * **Por qué:** Garantiza que la tarea obtenga una copia fresca y estable de las variables de la notificación (`%antitle(1)`, `%antext`, etc.).

* **4. Copiar Nombre Completo**
    * `Variables` → `Definir Variable`. Nombre: `%nombre_para_split`, A: `%antitle(1)`.
    * **Por qué:** Creamos una copia segura para trabajar sobre ella, preservando la original.

* **5. Extraer Primer Nombre**
    * `Variables` → `Variable Split`. Nombre: `%nombre_para_split`, Divisor: ` ` (un espacio).
    * **Por qué:** Separa el nombre completo. El primer nombre queda en `%nombre_para_split1` para un saludo más personal.

* **6-10. Variables de Configuración y Prompt Dinámico**
    * **6. `Definir Variable`**: `%timeout_sesion_segundos`, A: `900`.
    * **7. `Definir Variable`**: `%ruta_historiales`, A: `Tasker/ChatHistory`.
    * **8. `Definir Variable`**: `%prompt_sistema`, A: *Pega aquí tu prompt en formato JSON (ver ejemplo abajo)*.
    * **9. `Definir Variable`**: `%prompt_dinamico`, A: `%prompt_sistema`.
    * **10. `Búsqueda Reemplazar Variable`**: Variable: `%prompt_dinamico`, Búsqueda: `%%USERNAME%%`, Reemplazar Con: `%nombre_para_split1`.
    * **Por qué:** Separamos la configuración del código. Creamos una copia del prompt y la personalizamos dinámicamente.

#### Fase 2: Gestión de la Memoria (Archivos de Historial)
* **11. Limpiar Nombre para Archivo**
    * `Variables` → `Búsqueda Reemplazar Variable`. Variable: `%antitle(1)`, Búsqueda: `\W`, Marcar `Regex`, Reemplazar Con: `_`.
    * **Por qué:** Elimina caracteres especiales del nombre para crear un nombre de archivo válido.

* **12. Construir Ruta del Archivo**
    * `Variables` → `Definir Variable`. Nombre: `%archivo_historial`, A: `%ruta_historiales/%antitle(1)_history.json`.

* **13. Inicializar Variable de Historial**
    * `Variables` → `Definir Variable`. Nombre: `%historial_json_raw`, A: `[]`.
    * **Por qué:** Asegura que la variable siempre exista, previniendo errores para contactos nuevos.

* **14-22. Lógica de Timeout y Lectura del Historial (Bloque Corregido)**
    * **14. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Existe`, Guardar En: `%historial_existe`.
    * **15. `Si`**: `%historial_existe` `~` `true`.
        * **16. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Fecha de Modificación`, Guardar En: `%fecha_ultimo_mensaje`.
        * **17. `Definir Variable`**: `%segundos_desde_ultimo`, A: `%TIMES - %fecha_ultimo_mensaje`, Marcar `Hacer Cuentas`.
        * **18. `Si`**: `%segundos_desde_ultimo > %timeout_sesion_segundos`.
            * **19. `Borrar Fichero`**: Ruta: `%archivo_historial`, Marcar `Continuar Tarea Tras Error`.
        * **20. `Si no`** (Else).
            * **21. `Leer Fichero`**: Fichero: `%archivo_historial`, A Variable: `%historial_json_raw`.
        * **22. `Fin Si`**.
    * **23. `Fin Si`**
    * **Por qué:** Esta estructura es robusta. Comprueba si el archivo existe. Si existe, comprueba su antigüedad: si es viejo lo borra, si no es viejo lo lee. El `Fin Si` final (#23) cierra correctamente el bloque que comenzó en la acción #15.

#### Fase 3: Comunicación con la IA
* **24. Preparar Envío (JavaScriptlet)**
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

* **25. Petición HTTP**
    * `Red` → `Petición HTTP`.
    * **Configuración:** Método `POST`, URL `https://api.openai.com/v1/chat/completions`, Cabeceras con `%AI_API_KEY`, Cuerpo con `%messages_array_json`, Timeout `60s`, Marcar `Confía en cualquier certificado` y **`Continuar Tarea Tras Error`**.
    * **Por qué "Continuar Tras Error":** Para asegurar que, incluso si la conexión a internet falla, la tarea llegue al final y libere el cerrojo.

* **26. Procesar Respuesta (JavaScriptlet)**
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
* **27. Guardar Historial Actualizado**
    * `Fichero` → `Escribir Fichero`.
    * **Configuración:** Fichero: `%archivo_historial`, Texto: `%new_history_to_save`.
    * **Por qué:** La acción crucial que da "memoria" al asistente.

* **28. Responder Mensaje**
    * `Plugin` → `AutoNotification` → `Reply`.
    * **Configuración:** Reply Text: `%gpt_reply`, Reply Action ID: `%anreplyaction(1)`, Marcar `Cancel Before Replying` y **`Continuar Tarea Tras Error`**.
    * **Por qué "Continuar Tras Error":** Si la notificación original ya no existe, esta acción fallará. Necesitamos que la tarea continúe para liberar el cerrojo.

* **29. Liberar Cerrojo (Limpieza Final)**
    * `Variables` → `Borrar Variable`.
    * **Nombre:** `%AutopilotIsRunning`.
    * **Por qué:** "Abre la puerta" para que el siguiente mensaje en la cola pueda ser procesado.

* **30. Manejo de Colisiones (Propiedades de Tarea)**
    * Dentro de la tarea, toca el **icono de engranaje ⚙️**.
    * En **Manejo de Colisiones**, selecciona `Abortar Tarea Nueva`.
    * **Por qué:** Como tu versión de Tasker no tiene `Ejecutar en Secuencia`, esta es la opción más segura. Junto con nuestro cerrojo manual, si llegan dos mensajes al mismo tiempo, el primero pasará y el segundo será ignorado, evitando un error. Es una limitación necesaria para asegurar la estabilidad del sistema.

### ✍️ Paso 4: Ejemplo de Prompt para el Asistente

Puedes usar esta plantilla como base para tu variable `%prompt_sistema`. Recuerda que `%%USERNAME%%` será reemplazado automáticamente por el primer nombre del contacto.

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
