# Asistente para Choferes 🚚

## 👤 Autor

**Javier Castillo Millán**  
Clave: 169589  
Proyecto desarrollado como parte de la materia de Temas Selectos de Análisis de Datos del ITAM 2025

## Descripción

Este proyecto implementa un asistente virtual especializado para choferes de transporte, utilizando Azure OpenAI para proporcionar información inmediata sobre normativas, reglamentos y procedimientos ante situaciones como detenciones por autoridades viales.

- El notebook funcionando se encuentra en: [assistant.ipynb](./assistant.ipynb)
- La presentación se encuentra en: [Presentación](./Presentación.pdf)
- El vídeo se encuentra en: [Vídeo](./Vídeo.mp4)
- Para un tutorial sobre como configurar azure: [Tutorial Azure](./tutorial-azure-config.md)
- Un ejemplo de acceso a Azure de forma segura con autenticación: [AZURE AI FOUNDRY.ipynb](./AZURE%20AI%20FOUNDRY.ipynb)

## 📋 Índice

- [Asistente para Choferes 🚚](#asistente-para-choferes-)
  - [👤 Autor](#-autor)
  - [Descripción](#descripción)
  - [📋 Índice](#-índice)
  - [🔍 Descripción General](#-descripción-general)
  - [✨ Funcionalidades](#-funcionalidades)
  - [⚙️ Configuración del Entorno](#️-configuración-del-entorno)
  - [🏗️ Estructura del Código](#️-estructura-del-código)
    - [1. Inicialización del Cliente de Azure OpenAI](#1-inicialización-del-cliente-de-azure-openai)
    - [2. Definición del Papel del Asistente](#2-definición-del-papel-del-asistente)
    - [3. Funciones para Consulta y Formateo](#3-funciones-para-consulta-y-formateo)
      - [Función Principal de Consulta](#función-principal-de-consulta)
      - [Formateo de Respuestas](#formateo-de-respuestas)
  - [🔄 Interacción con Azure Cognitive Search](#-interacción-con-azure-cognitive-search)
  - [🔄 Creación de Fuente de Conocimiento en Azure AI Foundry](#-creación-de-fuente-de-conocimiento-en-azure-ai-foundry)
    - [1. Preparación de documentos](#1-preparación-de-documentos)
    - [2. Creación del índice en Azure AI Search](#2-creación-del-índice-en-azure-ai-search)
    - [3. Proceso de indexación](#3-proceso-de-indexación)
    - [4. Integración con el script](#4-integración-con-el-script)
  - [🔒 Ventajas de Seguridad con Azure AI Services](#-ventajas-de-seguridad-con-azure-ai-services)
    - [Guardrails y Protecciones](#guardrails-y-protecciones)
    - [Configuración específica en el proyecto](#configuración-específica-en-el-proyecto)
  - [🚀 Uso del Asistente](#-uso-del-asistente)
  - [🔧 Personalización](#-personalización)
    - [Modificar el Papel del Asistente](#modificar-el-papel-del-asistente)
    - [Configuración del Modelo](#configuración-del-modelo)

## 🔍 Descripción General

El Asistente para Choferes es una aplicación Jupyter Notebook que conecta a la API de Azure OpenAI para proporcionar respuestas precisas y contextualizadas. El asistente puede:

- Responder preguntas sobre normativas de transporte
- Explicar procedimientos ante detenciones por autoridades
- Proporcionar información sobre documentación requerida
- Citar fuentes específicas de documentos legales

La aplicación utiliza una base de conocimiento indexada en Azure Cognitive Search que incluye reglamentos oficiales, manuales de procedimientos y directrices de la empresa.

## ✨ Funcionalidades

- **Consulta basada en contexto**: El asistente analiza documentos como "Normas de seguridad.pdf", "Instructivo de contactos.pdf" y leyes de transporte.
- **Interfaz interactiva**: Implementada con Jupyter widgets para una experiencia de usuario fluida.
- **Historial de conversación**: Mantiene el contexto a través de múltiples consultas.
- **Referencias a documentos**: Incluye enlaces a los documentos fuente citados.
- **Estadísticas de uso**: Muestra métricas de tokens utilizados.

## ⚙️ Configuración del Entorno

El proyecto utiliza variables de entorno para las credenciales de Azure:

```python
from dotenv import load_dotenv
import os

# Cargar .env
load_dotenv()

sea_key = os.getenv("SEA_KEY")
sub_key = os.getenv("SUB_KEY")
endpoint_name = os.getenv("ENDPOINT_NAME")
deployment_name = os.getenv("DEPLOYMENT_NAME")
search_endpoint_name = os.getenv("SEARCH_ENDPOINT_NAME")
blob_sas_url_name = os.getenv("BLOB_SAS_URL_NAME")
```

Debes crear un archivo `.env` en el directorio raíz con las siguientes variables:

```{env}
SEA_KEY=tu_clave_de_azure_search
SUB_KEY=tu_clave_de_suscripcion_azure
ENDPOINT_NAME=https://tu-endpoint-openai.openai.azure.com
DEPLOYMENT_NAME=tu-deployment-model
SEARCH_ENDPOINT_NAME=https://tu-endpoint-search.search.windows.net
BLOB_SAS_URL_NAME=url_con_token_sas_para_acceso_a_documentos
```

## 🏗️ Estructura del Código

### 1. Inicialización del Cliente de Azure OpenAI

```python
client = AzureOpenAI(
    azure_endpoint=endpoint,
    api_key=subscription_key,
    api_version="2025-01-01-preview",
)
```

Este bloque configura el cliente que se comunicará con la API de Azure OpenAI, utilizando los parámetros definidos en el archivo de variables de entorno.

### 2. Definición del Papel del Asistente

El archivo `role.md` contiene las instrucciones detalladas para el comportamiento del asistente. Este papel define:

- El ámbito de las preguntas que puede responder
- El proceso de validación ética y legal
- El formato y estilo de las respuestas
- Las fuentes de conocimiento autorizadas

### 3. Funciones para Consulta y Formateo

#### Función Principal de Consulta

```python
def consultar_asistente(consulta, historial=None):
    if historial is None:
        historial = [{"role": "system", "content": system_message}]
    
    # Añadir la nueva consulta al historial
    historial.append({"role": "user", "content": consulta})
    
    # Generar la completación
    completion = client.chat.completions.create(
        model=deployment,
        messages=historial,
        max_tokens=2500,
        temperature=0.2,
        top_p=0.95,
        frequency_penalty=0,
        presence_penalty=0,
        extra_body={
          "data_sources": [{
              "type": "azure_search",
              "parameters": {
                "endpoint": search_endpoint,
                "index_name": "knowledgellmchoferes",
                "authentication": {
                  "type": "api_key",
                  "key": search_key
                },
                "query_type": "simple",
                "in_scope": True,
                "top_n_documents": 20
              }
            }]
        }
    )
    
    # Añadir la respuesta al historial
    respuesta = completion.choices[0].message.content
    historial.append({"role": "assistant", "content": respuesta})
    
    return completion, historial
```

Esta función:

1. Mantiene el historial de la conversación
2. Envía la consulta al modelo de lenguaje
3. Configura parámetros como temperatura (creatividad) y tokens máximos
4. Conecta a Azure Cognitive Search para obtener documentos relevantes
5. Añade la respuesta al historial de conversación

#### Formateo de Respuestas

```python
def generar_html_respuesta(completion):
    # Extraer el contenido de la respuesta
    content = completion.choices[0].message.content
    
    # Estandarizar las referencias en el texto [docN] -> [N]
    content = estandarizar_referencias(content)
    
    # Convertir markdown a HTML
    html_content = render_markdown(content)
    
    # Extraer las citas/fuentes utilizadas...
```

Esta función transforma la respuesta del modelo en HTML formateado para la interfaz, incluyendo referencias a documentos y estadísticas de uso.

## 🔄 Interacción con Azure Cognitive Search

El proyecto utiliza Azure Cognitive Search para indexar y consultar documentos relevantes:

```python
"data_sources": [{
    "type": "azure_search",
    "parameters": {
      "endpoint": search_endpoint,
      "index_name": "knowledgellmchoferes",
      "authentication": {
        "type": "api_key",
        "key": search_key
      },
      "query_type": "simple",
      "in_scope": True,
      "top_n_documents": 20
    }
  }]
```

Esta configuración:

- Conecta con el índice específico `knowledgellmchoferes`
- Utiliza autenticación por API key
- Configura la búsqueda para recuperar hasta 20 documentos relevantes
- Habilita la recuperación inteligente basada en la relevancia semántica

El sistema utiliza el contenedor de Azure Blob Storage para almacenar los documentos originales. Cuando el asistente cita una fuente, se genera una URL con token SAS para permitir la descarga directa del documento:

```python
def generar_url_documento(filepath):
    """
    Genera una URL para descargar un documento desde el blob storage
    """
    # Escapar caracteres especiales en el nombre del archivo
    import urllib.parse
    
    # Ajustar la ruta para incluir la carpeta "documentos/"
    filepath_completo = f"documentos/{filepath}"
    filename_encoded = urllib.parse.quote(filepath_completo)
    
    # Extraer la parte de la URL base y el token SAS
    container_url = blob_sas_url.split('?')[0]  # URL del contenedor sin el token
    sas_token = blob_sas_url.split('?')[1]      # Token SAS
    
    # Construir la URL completa
    document_url = f"{container_url}/{filename_encoded}?{sas_token}"
    
    return document_url
```

Esta función permite que los usuarios accedan a los documentos originales directamente desde la interfaz.

## 🔄 Creación de Fuente de Conocimiento en Azure AI Foundry

El proyecto utiliza una fuente de conocimiento creada e indexada a través de Azure AI Foundry, siguiendo estos pasos:

### 1. Preparación de documentos

Los documentos de normativas, leyes y manuales se almacenan en un contenedor de Azure Blob Storage, organizados en una estructura de carpetas lógica. Esto incluye:

- Normas de seguridad.pdf
- Instructivo de contactos.pdf
- Ley de caminos, puentes y autotransporte federal.pdf
- Reglamento de tránsito en carreteras y puentes de jurisdicción federal.pdf

### 2. Creación del índice en Azure AI Search

Para crear el índice de conocimiento:

1. Se accede al portal de Azure AI Foundry
2. Se crea un nuevo recurso de AI Search
3. Se configura un nuevo índice `knowledgellmchoferes` con las siguientes propiedades:
   - Campos para almacenar metadatos (título, autor, fecha)
   - Campo de contenido para el texto extraído
   - Campos para referencias y citas
   - Campo para la ruta del archivo original

### 3. Proceso de indexación

El proceso de indexación incluye:

1. **Extracción de texto**: Los PDFs son procesados mediante reconocimiento óptico de caracteres (OCR) cuando es necesario
2. **Chunking**: Los documentos largos se dividen en fragmentos manejables (chunks)
3. **Enriquecimiento semántico**: Se generan embeddings vectoriales para cada chunk
4. **Indexación**: Los chunks procesados se almacenan en el índice de Azure Cognitive Search

### 4. Integración con el script

El script interactúa con la fuente de conocimiento a través de varias capas:

```{text}
┌─────────────────┐    ┌──────────────────┐    ┌───────────────────┐
│                 │    │                  │    │                   │
│ Azure OpenAI    │◄───┤ Azure AI Search  │◄───┤ Azure Blob Storage│
│ (Procesamiento) │    │ (Índice)         │    │ (Documentos)      │
│                 │    │                  │    │                   │
└────────┬────────┘    └──────────────────┘    └───────────────────┘
         │
         ▼
┌────────────────────┐
│                    │
│ Aplicación Jupyter │
│ (Interfaz)         │
│                    │
└────────────────────┘
```

Cuando un usuario realiza una consulta:

1. La consulta se envía al modelo Azure OpenAI a través de la API
2. Azure OpenAI utiliza internamente Azure AI Search para encontrar chunks relevantes
3. El modelo genera una respuesta basada en los chunks recuperados
4. La respuesta incluye referencias a los documentos originales
5. El script genera URLs con tokens SAS para acceder a los documentos en Azure Blob Storage

La configuración `data_sources` en el código permite esta interacción fluida:

```python
"data_sources": [{
    "type": "azure_search",
    "parameters": {
      "endpoint": search_endpoint,
      "index_name": "knowledgellmchoferes",
      "authentication": {
        "type": "api_key",
        "key": search_key
      },
      "query_type": "simple",
      "in_scope": True,
      "top_n_documents": 20
    }
  }]
```

## 🔒 Ventajas de Seguridad con Azure AI Services

La implementación del asistente en Azure AI Services proporciona importantes ventajas de seguridad y control frente a otras alternativas:

### Guardrails y Protecciones

1. **Filtrado de contenido integrado**: Azure OpenAI implementa filtros que detectan y bloquean:
   - Solicitudes de actividades ilegales
   - Intentos de obtener información sensible
   - Generación de contenido dañino o discriminatorio

2. **Detección de prompt injection**: El servicio incluye protecciones contra:
   - Intentos de jailbreak del modelo
   - Prompt leaking (extraer el prompt del sistema)
   - Manipulación para eludir las restricciones del sistema

3. **Control de datos y privacidad**:
   - Los datos no se utilizan para entrenar modelos
   - Cumplimiento con GDPR y otras regulaciones
   - Posibilidad de implementación en regiones específicas

### Configuración específica en el proyecto

El sistema implementa varias capas de seguridad:

1. **Validación ética y legal**: El papel del asistente (definido en `role.md`) incluye instrucciones específicas para rechazar solicitudes no éticas:

```markdown
## Primero: Validación Ética y Legal
- ¿Se solicita o promueve una acción ilegal (sobornos, falsificación, evasión)?
  - Si sí, rechaza la consulta y explica que está prohibida por las políticas de la empresa.
  - Si no, continúa.
```

2. **Control de acceso**: El uso de tokens SAS para acceder a documentos garantiza que solo los usuarios autorizados puedan ver la información original.

3. **Parámetros de generación seguros**: La configuración del modelo con valores conservadores reduce la posibilidad de generación de respuestas problemáticas:

```python
temperature=0.2,  # Valor bajo para respuestas más deterministas
top_p=0.95,
frequency_penalty=0,
presence_penalty=0,
```

4. **Aislamiento de datos**: Toda la información se mantiene dentro del ecosistema de Azure, sin exposición a servicios de terceros.

Estas capas de protección hacen que la solución sea adecuada para entornos corporativos donde la seguridad, conformidad y control de la información son prioritarios.

El proyecto implementa una interfaz interactiva utilizando widgets de Jupyter:

```python
def interfaz_interactiva_con_historial():
    # Definir el CSS para la interfaz
    css = """
    <style>
        /* Estilos para la interfaz principal */
        .app-container {
            max-width: 800px;
            margin: 0 auto;
            font-family: Arial, sans-serif;
            overflow-x: hidden;
        }
        
        /* Contenedor de conversación con scroll vertical */
        .conversation-scroll {
            max-height: 600px;
            overflow-y: auto;
            overflow-x: hidden;
            border: 1px solid #eaeaea;
            border-radius: 10px;
            padding: 10px;
            margin-bottom: 20px;
            background-color: #ffffff;
            width: 100%;
            max-width: 780px;
        }
        ...
```

La interfaz incluye:

- Un encabezado de aplicación con título y descripción
- Área de conversación con desplazamiento vertical
- Campo de entrada de texto para preguntas
- Botones para enviar consultas y limpiar el historial
- Presentación de respuestas con formato mejorado

## 🚀 Uso del Asistente

Para usar el asistente:

1. Ejecuta todas las celdas del notebook para inicializar la interfaz
2. Escribe tu consulta en el campo de texto (por ejemplo: "¿Qué documentos debo presentar si me detiene la Guardia Nacional?")
3. Haz clic en "Enviar Consulta"
4. Revisa la respuesta del asistente, incluyendo:
   - Información detallada sobre la consulta
   - Referencias a documentos específicos
   - Enlaces para acceder a los documentos citados

## 🔧 Personalización

### Modificar el Papel del Asistente

Para cambiar el comportamiento del asistente, modifica el archivo `role.md`. Puedes ajustar:

- Fuentes de conocimiento
- Criterios de validación
- Estilo de respuesta
- Ejemplos de consultas aceptadas/rechazadas

### Configuración del Modelo

Para ajustar la generación de respuestas, modifica los parámetros en la función `consultar_asistente()`:

```python
completion = client.chat.completions.create(
    model=deployment,
    messages=historial,
    max_tokens=2500,  # Ajusta para respuestas más largas/cortas
    temperature=0.2,  # Aumenta para más creatividad, disminuye para más precisión
    top_p=0.95,       # Controla diversidad de tokens seleccionados
    frequency_penalty=0,  # Penaliza repeticiones
    presence_penalty=0,   # Penaliza temas ya mencionados
    ...
)
```
