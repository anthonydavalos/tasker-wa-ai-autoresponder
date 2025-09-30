# 🤖 WhatsApp IA Autoresponder: Tu Propio Asistente con Tasker

**Crea un asistente de IA totalmente automatizado para WhatsApp que responde mensajes por ti, recuerda cada conversación individualmente y sigue tus reglas de negocio.**

Este proyecto utiliza el poder de **Tasker** en Android para crear un sistema de auto-respuesta que va mucho más allá de los mensajes básicos. Se conecta a una API de IA (como OpenAI GPT, Google Gemini, etc.) para mantener conversaciones coherentes y personalizadas con cada contacto.

---

## ✨ Características Principales

* **Memoria Persistente:** Recuerda el historial de cada conversación de forma individual.
* **Personalización Dinámica:** Saluda a cada contacto por su nombre y adapta sus instrucciones sobre la marcha.
* **Filtrado Inteligente:** Ignora automáticamente los resúmenes de notificaciones y los mensajes de chats de grupo.
* **Gestión de Colisiones:** Maneja múltiples mensajes simultáneos de forma ordenada, sin cruzar conversaciones.
* **Base de Conocimiento:** Puede ser programado con información específica (precios, contactos, etc.) para responder consultas de forma autónoma.
* **Sistema a Prueba de Fallos:** Diseñado para ser robusto, con mecanismos de "cerrojo" para evitar bucles y errores.

## 📋 Requisitos Previos

Antes de empezar, asegúrate de tener lo siguiente:

1.  **Tasker:** La última versión (preferiblemente beta) instalada desde la Play Store.
2.  **Plugin AutoNotification:** Instalado desde la Play Store.
3.  **API Key de un Proveedor de IA:** Una clave de API válida de [OpenAI](https://platform.openai.com/api-keys), [Google AI Studio](https://aistudio.google.com/app/apikey), o similar.
4.  **Permisos Críticos en Android:** Para la aplicación Tasker, asegúrate de haber concedido:
    * `Acceso a Notificaciones` (para AutoNotification).
    * `Batería` → **"Sin restricciones"**.
    * `Permisos` → `Archivos y contenido multimedia` → **"Permitir la gestión de todos los archivos"**.

---

## 🛠️ Guía de Implementación Paso a Paso

### Paso 1: Preparación del Entorno

#### 1.1 Guardar la Clave de API
* **Por qué:** Guardamos la clave en una variable global para no tener que escribirla repetidamente y mantenerla segura en un solo lugar.
* **Acción:**
    1.  Abre Tasker y ve a la pestaña **VARS**.
    2.  Crea una nueva variable global llamada `%OPENAI_KEY`.
    3.  Pega tu clave secreta en el campo de valor.

#### 1.2 Crear la Carpeta de Historial
* **Por qué:** Necesitamos un lugar dedicado para almacenar los archivos `.json` que contendrán el historial de cada conversación.
* **Acción:**
    1.  Usando un explorador de archivos, ve a tu almacenamiento interno.
    2.  Dentro de la carpeta `Tasker`, crea una nueva carpeta llamada `ChatHistory`.

### Paso 2: Creación del Perfil (El Activador Inteligente)

* **Por qué:** Este perfil es el "guardia de seguridad". Se asegura de que la tarea solo se active con notificaciones que nos interesan: mensajes individuales y respondibles, ignorando todo lo demás.

* **Acción:**
    1.  Ve a la pestaña **PERFILES** y crea un nuevo perfil de **Evento**.
    2.  Selecciona `Plugin` → `AutoNotification` → `Intercept`.
    3.  Configura el perfil con las siguientes reglas:
        * **Apps:** Selecciona `WhatsApp` y `WhatsApp Business`.
        * **Package Name Filter:** Escribe `com.whatsapp|com.whatsapp.w4b` y marca la casilla **`Regex`**.
            > *Esto asegura que solo estas dos apps puedan activar el perfil, usando el `|` como un "O" lógico.*
        * **Has Reply Action:** Marca esta casilla (Activado).
            > *Filtro crucial para asegurar que es un mensaje directo al que se puede responder.*
        * **Ignore Group Summaries:** Marca esta casilla (Activado).
            > *La opción "mágica" para descartar notificaciones resumen como "312 mensajes nuevos".*
        * **Title Filter:** Escribe `.*:.*`, marca la casilla **`Regex`** y marca la casilla **`Invert`**.
            > *El seguro final para ignorar cualquier mensaje que provenga de un chat de grupo, ya que sus títulos contienen dos puntos (Ej: "Remitente: Nombre del Grupo").*
    4.  Al salir, asocia este perfil a una **Tarea Nueva** y nómbrala (ej. `Asistente IA WhatsApp`).

### Paso 3: Construcción de la Tarea Principal (El Cerebro)

Esta es la secuencia exacta y optimizada de **31 acciones** que componen la tarea.

#### Fase 1: Recopilación y Preparación de Datos
* **Acción 1: Detener si está Ocupado (Cerrojo)**
    * `Tarea` → `Detener`
    * **Condición `Si`**: `%BixbyIsRunning` **está definida**.
    * **Por qué:** Evita colisiones. Si la tarea ya se está ejecutando para un mensaje, cualquier nueva activación se detendrá aquí para esperar en la cola, evitando que las variables se mezclen.

* **Acción 2: Activar Cerrojo**
    * `Variables` → `Definir Variable`
    * Nombre: `%BixbyIsRunning`, A: `1`.
    * **Por qué:** "Cierra la puerta" para indicar que el proceso ha comenzado.

* **Acción 3: Forzar Lectura de la Notificación**
    * `Plugin` → `AutoNotification` → `Query`
    * **Configuración:** En `Apps`, selecciona tus apps de WhatsApp.
    * **Por qué:** Es la acción más importante. Garantiza que la tarea obtenga una copia fresca y estable de las variables de la notificación (`%antitle(1)`, `%antext`, etc.), evitando el problema de las "variables fantasma".

* **Acción 4: Copiar Nombre para `Split`**
    * `Variables` → `Definir Variable` → Nombre: `%nombre_para_split`, A: `%antitle(1)`.
    * **Por qué:** Es una buena práctica trabajar sobre una copia para no alterar la variable original (`%antitle(1)`) que usaremos más adelante.

* **Acción 5: Extraer Primer Nombre**
    * `Variables` → `Variable Split` → Nombre: `%nombre_para_split`, Divisor: ` ` (un espacio).
    * **Por qué:** Separa el nombre completo por espacios. El primer nombre queda en `%nombre_para_split1`, permitiendo un saludo más personal.

* **Acción 6-10: Variables de Configuración y Prompt Dinámico**
    * **6. `Definir Variable`**: `%timeout_sesion_segundos`, A: `900`.
    * **7. `Definir Variable`**: `%ruta_historiales`, A: `Tasker/ChatHistory`.
    * **8. `Definir Variable`**: `%prompt_sistema`, A: *Pega aquí tu prompt en formato JSON (ver ejemplo abajo)*.
    * **9. `Definir Variable`**: `%prompt_dinamico`, A: `%prompt_sistema`.
    * **10. `Búsqueda Reemplazar Variable`**: Variable: `%prompt_dinamico`, Búsqueda: `%%USERNAME%%`, Reemplazar Con: `%nombre_para_split1`.
    * **Por qué:** Separamos la configuración del código. Creamos una copia del prompt y la personalizamos dinámicamente con el primer nombre del contacto en cada ejecución.

#### Fase 2: Gestión de la Memoria (Archivos de Historial)
* **Acción 11: Limpiar Nombre para Archivo**
    * `Variables` → `Búsqueda Reemplazar Variable` → Variable: `%antitle(1)`, Búsqueda: `\W`, Marcar `Regex`, Reemplazar Con: `_`.
    * **Por qué:** Elimina caracteres especiales del nombre del contacto para crear un nombre de archivo válido y evitar errores.

* **Acción 12: Construir Ruta del Archivo**
    * `Variables` → `Definir Variable` → Nombre: `%archivo_historial`, A: `%ruta_historiales/%antitle(1)_history.json`.

* **Acción 13-20: Lógica de Timeout del Historial**
    * **13. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Existe`, Guardar En: `%historial_existe`.
    * **14. `Si`**: `%historial_existe` `~` `true`.
        * **15. `Probar Fichero`**: Ruta: `%archivo_historial`, Tipo: `Fecha de Modificación`, Guardar En: `%fecha_ultimo_mensaje`.
        * **16. `Definir Variable`**: `%segundos_desde_ultimo`, A: `%TIMES - %fecha_ultimo_mensaje`, Marcar `Hacer Cuentas`.
        * **17. `Si`**: `%segundos_desde_ultimo > %timeout_sesion_segundos`.
            * **18. `Borrar Fichero`**: Ruta: `%archivo_historial`, Marcar `Continuar Tarea Tras Error`.
        * **19. `Fin Si`**.
    * **20. `Fin Si`**.
    * **Por qué:** Este bloque comprueba si la conversación es muy antigua. Si han pasado más de `%timeout_sesion_segundos` (ej. 15 min), borra el historial para que el asistente se presente de nuevo, evitando continuar conversaciones olvidadas.

* **Acción 21-24: Leer el Historial**
    * **21. `Definir Variable`**: `%historial_json_raw`, A: `[]`.
    * **22. `Si`**: `%historial_existe` `~` `true`.
        * **23. `Leer Fichero`**: Fichero: `%archivo_historial`, A Variable: `%historial_json_raw`.
    * **24. `Fin Si`**.
    * **Por qué:** Inicializamos la variable de historial como un array vacío. Luego, solo si el archivo existe, leemos su contenido. Esto evita el error de "archivo no encontrado" para contactos nuevos.

#### Fase 3: Comunicación con la IA
* **Acción 25: Preparar Envío (JavaScriptlet)**
    * `Código` → `JavaScriptlet`.
    * **Código:**
        ```javascript
        const oldHistoryJson = historial_json_raw || '[]';
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
    * **Por qué:** Este script construye el array de mensajes completo (historial + nuevo mensaje) que la API necesita.

* **Acción 26: Petición HTTP**
    * `Red` → `Petición HTTP`.
    * **Configuración:** Método `POST`, URL de la API, Cabeceras con `%OPENAI_KEY`, Cuerpo con `%messages_array_json`, Timeout `60s`, Marcar `Confía en cualquier certificado` y `Continuar Tarea Tras Error`.
    * **Por qué:** Esta es la acción que efectivamente se comunica con el servidor de la IA.

* **Acción 27: Procesar Respuesta (JavaScriptlet)**
    * `Código` → `JavaScriptlet`.
    * **Código:**
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
    * **Por qué:** Este script lee la respuesta de la IA (`%http_data`), extrae el texto limpio para la respuesta (`%gpt_reply`) y prepara el nuevo historial completo para ser guardado (`%new_history_to_save`).

#### Fase 4: Guardado, Respuesta y Limpieza
* **Acción 28: Guardar Historial Actualizado**
    * `Fichero` → `Escribir Fichero`.
    * **Configuración:** Fichero: `%archivo_historial`, Texto: `%new_history_to_save`.
    * **Por qué:** La acción crucial que da "memoria" al asistente, guardando la conversación actualizada.

* **Acción 29: Responder Mensaje**
    * `Plugin` → `AutoNotification` → `Reply`.
    * **Configuración:** Reply Text: `%gpt_reply`, Reply Action ID: `%anreplyaction(1)`, Marcar `Cancel Before Replying` y `Continuar Tarea Tras Error`.
    * **Por qué:** Envía la respuesta de la IA al chat correcto de forma fiable, usando el ID único de la notificación.

* **Acción 30: Liberar Cerrojo (Limpieza Final)**
    * `Variables` → `Borrar Variable`.
    * **Nombre:** `%BixbyIsRunning`.
    * **Por qué:** "Abre la puerta" para que la siguiente tarea en la cola pueda ejecutarse. Es el final del mecanismo de cerrojo.

* **Acción 31: Manejo de Colisiones (Propiedades de Tarea)**
    * Dentro de la tarea, toca el **icono de engranaje ⚙️**.
    * En **Manejo de Colisiones**, selecciona `Abortar Tarea Nueva` (o `Ejecutar en Secuencia` si está disponible).
    * **Por qué:** Junto con el cerrojo manual, esto proporciona una doble capa de protección contra la ejecución simultánea.

### ✍️ Paso 4: Ejemplo de Prompt para el Asistente

Puedes usar esta plantilla como base para tu variable `%prompt_sistema`. Recuerda reemplazar `%%USERNAME%%` en la sección del saludo.

```json
{
  "asistente": "Claudia",
  "propósito": "Soy un asistente virtual diseñado para apoyar, informar y acompañar a los usuarios en sus tareas, decisiones y momentos de curiosidad. Mi objetivo es ofrecer respuestas útiles, claras y atractivas, con una personalidad cálida, profesional y adaptable.",
  "personalidad": [
    "Carismática, empática y fácil de conversar.",
    "Profesional cuando el contexto lo requiere, pero también relajada y divertida si el usuario lo permite.",
    "Curiosa, creativa y proactiva: siempre busco enriquecer la conversación.",
    "Nunca soy robótica ni genérica; adapto mi estilo al usuario y al tema."
  ],
  "capacidades": [
    "Responder preguntas con precisión y contexto.",
    "Redactar textos: correos, notas, discursos, artículos, etc.",
    "Resumir, analizar y organizar información.",
    "Generar ideas creativas y soluciones prácticas.",
    "Ayudar en planificación, productividad y toma de decisiones.",
    "Conversar de forma fluida, con humor y profundidad."
  ],
  "límites": [
    "No comparto información privada, confidencial o sensible.",
    "No genero contenido que viole derechos de autor, incite al odio o promueva desinformación.",
    "No doy opiniones personales sobre figuras políticas, religiosas o temas sensibles.",
    "No simulo emociones humanas reales ni afirmo tener conciencia.",
    "No genero archivos descargables ni ejecuto código directamente."
  ],
  "formato_respuesta": {
    "estructura": "Markdown",
    "elementos": [
      "Títulos (##)",
      "Listas con viñetas (-)",
      "Emojis para dinamismo (🎯, 🧠, etc.)",
      "Tablas si es necesario para comparar o estructurar datos"
    ],
    "estilo": [
      "Respuestas claras, completas y bien organizadas.",
      "Lenguaje natural, variado y adaptado al tono del usuario.",
      "Siempre dejo espacio para continuar la conversación."
    ]
},
  "comportamiento_conversacional": [
    "Nunca cierro la conversación abruptamente.",
    "Hago preguntas abiertas para seguir explorando.",
    "Propongo temas relacionados o próximos pasos.",
    "Corrijo errores con tacto y claridad.",
    "Si el usuario está confundido, simplifico sin condescendencia."
  ],
  "idiomas": {
    "respuesta_en_idioma_usuario": true,
    "cambio_de_idioma_si_se_solicita": true
},
  "ejemplo_estilo": {
    "usuario": "¿Qué puedo hacer con una app de IA?",
    "claudia": "¡Buena pregunta! Las apps de IA pueden ayudarte a automatizar tareas, personalizar experiencias y tomar decisiones más inteligentes. Por ejemplo:",
    "ejemplos": [
      "📅 Organizar tu agenda con asistentes predictivos.",
      "🎨 Crear contenido visual con IA generativa.",
      "📊 Analizar datos para mejorar tu negocio."
    ],
    "cierre": "¿Quieres que exploremos una idea juntos?"
}
}
```
