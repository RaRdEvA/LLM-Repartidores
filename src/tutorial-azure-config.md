# Tutorial: Configuración de Azure Cognitive Search y Azure OpenAI para Asistentes Basados en Conocimiento

Este tutorial detalla el proceso para configurar un sistema de Recuperación Aumentada por Generación (RAG) utilizando servicios de Azure, incluyendo la creación de índices en Azure Cognitive Search y la configuración de un endpoint de Azure OpenAI.

## Índice
- [Requisitos Previos](#requisitos-previos)
- [Parte 1: Creación de un Servicio Azure Cognitive Search](#parte-1-creación-de-un-servicio-azure-cognitive-search)
- [Parte 2: Preparación y Carga de Documentos](#parte-2-preparación-y-carga-de-documentos)
- [Parte 3: Creación de un Índice en Azure Cognitive Search](#parte-3-creación-de-un-índice-en-azure-cognitive-search)
- [Parte 4: Creación de un Recurso Azure OpenAI](#parte-4-creación-de-un-recurso-azure-openai)
- [Parte 5: Despliegue de un Modelo en Azure OpenAI](#parte-5-despliegue-de-un-modelo-en-azure-openai)
- [Parte 6: Configuración del Endpoint de Azure OpenAI](#parte-6-configuración-del-endpoint-de-azure-openai)
- [Parte 7: Integración de Azure Cognitive Search con Azure OpenAI](#parte-7-integración-de-azure-cognitive-search-con-azure-openai)
- [Parte 8: Prueba y Verificación](#parte-8-prueba-y-verificación)
- [Consideraciones Adicionales](#consideraciones-adicionales)

## Requisitos Previos

Para seguir este tutorial, necesitarás:

1. Una suscripción activa de Azure
2. Permisos de propietario o colaborador en la suscripción
3. Documentos para indexar (PDFs, Word, texto, etc.)
4. Azure CLI instalado (opcional, para operaciones avanzadas)
5. Conocimientos básicos de Azure y procesamiento de lenguaje natural

## Parte 1: Creación de un Servicio Azure Cognitive Search

### Paso 1: Acceder al Portal de Azure
1. Navega a [portal.azure.com](https://portal.azure.com)
2. Inicia sesión con tus credenciales

### Paso 2: Crear un nuevo recurso de Azure Cognitive Search
1. Haz clic en "Crear un recurso"
2. Busca "Azure Cognitive Search" y selecciónalo
3. Haz clic en "Crear"

### Paso 3: Configurar el servicio de búsqueda
1. **Detalles básicos**:
   - **Suscripción**: Selecciona tu suscripción de Azure
   - **Grupo de recursos**: Crea un nuevo grupo o selecciona uno existente
   - **Nombre del servicio**: Ingresa un nombre único (e.g., `knowledge-choferes-search`)
   - **Ubicación**: Elige la región más cercana a tus usuarios (e.g., Centro de EE. UU. Sur)
   - **Plan de tarifa**: Selecciona según tus necesidades (Basic es suficiente para pruebas, Standard para producción)

2. Haz clic en "Revisar + crear" y luego en "Crear"
3. Espera a que se complete la implementación (puede tardar unos minutos)

### Paso 4: Obtener las claves de acceso
1. Ve al recurso de Azure Cognitive Search recién creado
2. En el menú lateral, selecciona "Claves"
3. Guarda la **Clave de administración principal** (necesaria para administrar el servicio)
4. Guarda la **Cadena de conexión** para usarla más adelante
5. Anota también el **Punto de conexión del servicio** (URL)

## Parte 2: Preparación y Carga de Documentos

### Paso 1: Crear un servicio de Azure Blob Storage
1. En el Portal de Azure, haz clic en "Crear un recurso"
2. Busca "Cuenta de almacenamiento" y selecciónala
3. Configura los detalles básicos:
   - **Grupo de recursos**: Usa el mismo grupo creado anteriormente
   - **Nombre de la cuenta**: Ingresa un nombre único (e.g., `docstoragechoferes`)
   - **Ubicación**: Usa la misma región que tu servicio de búsqueda
   - **Rendimiento**: Estándar
   - **Tipo de cuenta**: StorageV2
   - **Replicación**: LRS (Localmente redundante)

4. Haz clic en "Revisar + crear" y luego en "Crear"
5. Espera a que se complete la implementación

### Paso 2: Crear un contenedor para tus documentos
1. Ve a tu cuenta de almacenamiento recién creada
2. En el menú lateral, selecciona "Contenedores"
3. Haz clic en "+ Contenedor"
4. Ingresa un nombre (e.g., `documentos`)
5. Establece el nivel de acceso público en "Privado"
6. Haz clic en "Crear"

### Paso 3: Cargar documentos
1. Selecciona el contenedor recién creado
2. Haz clic en "Cargar"
3. Selecciona los documentos desde tu computadora:
   - Normas de seguridad.pdf
   - Instructivo de contactos.pdf
   - Ley de caminos, puentes y autotransporte federal.pdf
   - Reglamento de tránsito en carreteras y puentes de jurisdicción federal.pdf
4. Haz clic en "Cargar"

### Paso 4: Generar un token SAS para acceso
1. En tu cuenta de almacenamiento, ve a "Firmas de acceso compartido" en el menú lateral
2. Configura los permisos:
   - **Servicios permitidos**: Blob
   - **Tipos de recursos permitidos**: Contenedor, Objeto
   - **Permisos**: Lectura, Lista
   - **Fechas de inicio y vencimiento**: Establece una duración apropiada (p.ej., 1 año)
3. Haz clic en "Generar SAS y cadena de conexión"
4. Copia y guarda la **URL de SAS de Blob** para usarla en tu aplicación

## Parte 3: Creación de un Índice en Azure Cognitive Search

### Paso 1: Crear un indexador de habilidades cognitivas
1. Ve a tu servicio Azure Cognitive Search
2. En el menú lateral, selecciona "Importar datos"
3. En la fuente de datos, selecciona "Azure Blob Storage"
4. Completa la información de conexión:
   - **Nombre de la fuente de datos**: `docs-source`
   - **Cadena de conexión**: Usa la cadena de conexión de tu cuenta de almacenamiento
   - **Contenedor**: Selecciona el contenedor `documentos` creado anteriormente
   - **Modo de análisis de blobs**: Usa "Texto" para la mayoría de los documentos

5. Haz clic en "Siguiente: Agregar habilidades cognitivas (opcional)"

### Paso 2: Configurar el conjunto de habilidades cognitivas
1. Selecciona "Adjuntar Cognitive Services" y elige un servicio existente o crea uno nuevo
2. En "Agregar enriquecimientos":
   - Selecciona "Extraer nombres de personas", "Extraer frases clave", "Detectar idioma"
   - Habilita "Habilitar OCR" para extraer texto de imágenes en PDFs
   - En "Guardar enriquecimientos en un almacén de conocimiento", marca "Proyecciones de imágenes" y "Documentos"

3. Haz clic en "Siguiente: Personalizar índice de destino"

### Paso 3: Configurar el índice
1. Establece el nombre del índice como `knowledgellmchoferes`
2. Selecciona una clave (`metadata_storage_path` suele ser buena opción)
3. Configura los campos del índice:
   - Asegúrate de que los campos importantes sean recuperables y consultables
   - Añade campos específicos para tu caso de uso (e.g., `chunk_id`, `filepath`, `title`)
   - Establece el perfil de puntuación si es necesario

4. Haz clic en "Siguiente: Crear un indexador"

### Paso 4: Configurar el indexador
1. Establece el nombre del indexador como `choferes-indexer`
2. Establece la programación según tus necesidades:
   - Para una única ejecución: Selecciona "Una vez"
   - Para actualizaciones periódicas: Configura un horario (e.g., diario)
   
3. Expande las "Opciones avanzadas":
   - **Tamaño del lote**: 1000 (predeterminado)
   - **Máximo de documentos por valor**: -1 (sin límite)
   - **Truncamiento del campo de imagen**: Deshabilitar

4. Haz clic en "Enviar" para crear e iniciar el indexador

### Paso 5: Supervisar el proceso de indexación
1. Desde tu servicio de búsqueda, ve a "Indexadores"
2. Selecciona tu indexador `choferes-indexer`
3. Revisa el historial de ejecución y el estado
4. Espera a que el estado cambie a "Correcto"

## Parte 4: Creación de un Recurso Azure OpenAI

### Paso 1: Solicitar acceso a Azure OpenAI (si aún no lo tienes)
1. Ve a [https://aka.ms/oai/access](https://aka.ms/oai/access)
2. Completa el formulario de solicitud para tu organización
3. Espera la aprobación (puede tardar varios días)

### Paso 2: Crear un recurso de Azure OpenAI
1. En el Portal de Azure, haz clic en "Crear un recurso"
2. Busca "Azure OpenAI" y selecciónalo
3. Haz clic en "Crear"

### Paso 3: Configurar el recurso de Azure OpenAI
1. **Detalles básicos**:
   - **Suscripción**: Selecciona tu suscripción de Azure
   - **Grupo de recursos**: Usa el mismo grupo creado anteriormente
   - **Región**: Elige una región compatible con Azure OpenAI (e.g., Este de EE. UU.)
   - **Nombre**: Ingresa un nombre único (e.g., `choferes-openai`)
   - **Plan de tarifa**: Standard S0

2. Haz clic en "Revisar + crear" y luego en "Crear"
3. Espera a que se complete la implementación

## Parte 5: Despliegue de un Modelo en Azure OpenAI

### Paso 1: Acceder a Azure OpenAI Studio
1. Ve a tu recurso de Azure OpenAI
2. Haz clic en "Ir a Azure OpenAI Studio" o navega a [https://oai.azure.com/](https://oai.azure.com/)
3. Selecciona tu directorio, suscripción y recurso de Azure OpenAI

### Paso 2: Implementar un modelo
1. En el panel izquierdo, selecciona "Implementaciones"
2. Haz clic en "Crear nueva implementación"
3. Selecciona un modelo:
   - Para asistentes basados en conocimiento, elige un modelo GPT-4 o GPT-3.5 Turbo (e.g., `gpt-4-1106-preview`)
   - Para casos de alta precisión, considera `gpt-4-32k`

4. Dale un nombre a tu implementación (e.g., `choferes-deployment`)
5. Ajusta la opción de filtrado de contenido según tus necesidades
6. Haz clic en "Crear"

### Paso 3: Obtener detalles del endpoint
1. En el recurso de Azure OpenAI, ve a "Claves y punto de conexión"
2. Guarda estos valores para usarlos en tu aplicación:
   - **Punto de conexión**: URL del endpoint
   - **Clave 1** o **Clave 2**: Clave de API para autenticación
   - **Nombre de implementación**: El nombre que asignaste (e.g., `choferes-deployment`)

## Parte 6: Configuración del Endpoint de Azure OpenAI

### Paso 1: Configurar las variables de entorno
En tu aplicación, necesitarás configurar las siguientes variables de entorno:

```
ENDPOINT_NAME=https://choferes-openai.openai.azure.com
DEPLOYMENT_NAME=choferes-deployment
SUB_KEY=tu-clave-de-api-azure-openai
```

### Paso 2: Inicializar el cliente de Azure OpenAI en tu código

```python
from openai import AzureOpenAI

# Cargar variables
endpoint = endpoint_name  # De tus variables de entorno
deployment = deployment_name  # De tus variables de entorno
subscription_key = sub_key  # De tus variables de entorno

# Inicializar el cliente de Azure OpenAI
client = AzureOpenAI(
    azure_endpoint=endpoint,
    api_key=subscription_key,
    api_version="2025-01-01-preview",  # Ajusta según la versión actual
)
```

## Parte 7: Integración de Azure Cognitive Search con Azure OpenAI

### Paso 1: Configurar las variables de entorno para Azure Cognitive Search
Añade estas variables de entorno a tu aplicación:

```
SEARCH_ENDPOINT_NAME=https://knowledge-choferes-search.search.windows.net
SEA_KEY=tu-clave-de-administracion-azure-search
```

### Paso 2: Integrar Azure Cognitive Search en tus llamadas a la API de OpenAI

```python
# Configurar la llamada a la API con integración de búsqueda
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
```

### Paso 3: Acceder a las citas y referencias de documentos
Las respuestas de la API incluirán metadatos sobre los documentos utilizados para generar la respuesta:

```python
# Extraer referencias de la estructura de diccionario
if (completion_dict.get('choices') and len(completion_dict['choices']) > 0 and
    completion_dict['choices'][0].get('message') and
    completion_dict['choices'][0]['message'].get('context') and
    completion_dict['choices'][0]['message']['context'].get('citations')):
    
    citations = completion_dict['choices'][0]['message']['context']['citations']
    
    for citation in citations:
        if citation.get('filepath'):
            referencias.append({
                'filepath': citation.get('filepath', ''),
                'chunk_id': citation.get('chunk_id', ''),
                'title': citation.get('title', '')
            })
```

## Parte 8: Prueba y Verificación

### Paso 1: Probar la búsqueda en Azure Cognitive Search
1. Ve a tu servicio de Azure Cognitive Search
2. Selecciona "Explorador de búsqueda"
3. Selecciona tu índice `knowledgellmchoferes`
4. Realiza algunas consultas de prueba para verificar que tus documentos estén indexados correctamente
5. Revisa los resultados y ajusta la configuración del índice si es necesario

### Paso 2: Probar la integración con Azure OpenAI
1. Crea un script de prueba simple que utilice el cliente de Azure OpenAI con la integración de búsqueda
2. Realiza algunas consultas de prueba relacionadas con tu dominio específico
3. Verifica que las respuestas incluyan información de tus documentos
4. Comprueba que las citas y referencias sean precisas

### Paso 3: Ajustar la configuración
Basándote en los resultados de las pruebas, ajusta:
- El número de documentos recuperados (`top_n_documents`)
- La temperatura del modelo (valores más bajos para respuestas más deterministas)
- Los campos incluidos en la búsqueda
- Los filtros de contenido y seguridad

## Consideraciones Adicionales

### Optimización de Costos
- El servicio Azure Cognitive Search se factura por hora según el nivel elegido
- Azure OpenAI se factura por tokens (entrada y salida)
- Considera implementar límites de uso y monitoreo de costos

### Seguridad
- Implementa control de acceso basado en roles (RBAC) para tus recursos
- Usa Azure Key Vault para almacenar claves y secretos
- Considera Private Link para conexiones privadas a tus servicios

### Mantenimiento y Actualizaciones
- Programa actualizaciones periódicas del índice de búsqueda
- Mantén tus documentos actualizados en el almacenamiento de blobs
- Supervisa el rendimiento y ajusta la configuración según sea necesario

### Mejoras de la Experiencia del Usuario
- Proporciona retroalimentación sobre los documentos utilizados
- Implementa un sistema de valoración para mejorar el modelo con el tiempo
- Considera agregar características como filtrado por fuentes o tipos de documentos

---

Este tutorial te ha guiado a través del proceso completo de configuración de un sistema RAG utilizando Azure Cognitive Search y Azure OpenAI. Estos servicios proporcionan una base sólida para crear asistentes inteligentes basados en conocimiento que pueden responder preguntas específicas de dominio con alta precisión y citar sus fuentes de manera transparente.
