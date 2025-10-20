# Análisis y Validación de Workflows N8N
## Sistema Agéntico de Atención al Cliente por WhatsApp

---

## Workflow 1: Clasificación de Mensajes
**Archivo:** `workflow-1-clasificacion-mensajes.json`

### Estado: ✅ VÁLIDO CON OBSERVACIONES

### Análisis Estructural:
- ✅ Estructura JSON correcta
- ✅ Campos requeridos presentes (name, nodes, connections, settings)
- ✅ 15 nodos correctamente definidos
- ✅ Conexiones bien establecidas

### Nodos Analizados:
1. **Webhook WhatsApp** (n8n-nodes-base.webhook)
   - ✅ Configurado como POST
   - ✅ Path: `whatsapp-webhook`
   - ✅ Response mode configurado

2. **Extraer Datos del Mensaje** (n8n-nodes-base.set)
   - ✅ Extrae: phoneNumber, message, timestamp, messageId
   - ⚠️ OBSERVACIÓN: Asume estructura `$json.body.from` - verificar formato de WhatsApp API

3. **Obtener Contexto de Conversación** (n8n-nodes-base.redis)
   - ✅ Operación GET correcta
   - ✅ Key dinámico basado en phoneNumber
   - ⚠️ ADVERTENCIA: Requiere configuración de credenciales Redis

4. **Chat Model - OpenAI** (@n8n/n8n-nodes-langchain.lmChatOpenAi)
   - ✅ Modelo: gpt-4o
   - ✅ Temperature: 0.3 (apropiado para clasificación)
   - ✅ Conexión AI correcta al agente

5. **Window Buffer Memory** (@n8n/n8n-nodes-langchain.memoryBufferWindow)
   - ✅ SessionKey: phoneNumber
   - ✅ Context window: 10 mensajes
   - ✅ Conexión AI correcta al agente

6. **AI Agent Clasificador** (@n8n/n8n-nodes-langchain.agent)
   - ✅ Prompt bien definido con categorías claras
   - ✅ Categorías: productos, pedidos, soporte_complejo, otro
   - ✅ Output parser habilitado
   - ✅ Conectado a Chat Model y Memory

7. **Switch por Clasificación** (n8n-nodes-base.switch)
   - ✅ 3 condiciones + fallback
   - ✅ Case insensitive
   - ✅ Outputs renombrados correctamente

8-11. **HTTP Requests a otros workflows**
   - ✅ Método POST correcto
   - ✅ Body parameters bien estructurados
   - ⚠️ OBSERVACIÓN: URLs localhost - cambiar en producción

12. **Guardar Contexto** (n8n-nodes-base.redis)
   - ✅ Operación SET correcta
   - ✅ TTL: 3600 segundos (1 hora)
   - ✅ Serializa JSON correctamente

13. **Responder Webhook** (n8n-nodes-base.respondToWebhook)
   - ✅ JSON response
   - ✅ Estructura correcta

### Problemas Detectados:
1. ⚠️ **CRÍTICO**: Los nodos de AI Tools (Chat Model, Memory) NO están conectados físicamente en el JSON
   - Falta la conexión `ai_languageModel` y `ai_memory` al AI Agent
   - **Estado actual**: Las conexiones SÍ están definidas en líneas 337-357
   - ✅ CORREGIDO: Las conexiones AI están presentes

2. ⚠️ **IMPORTANTE**: Falta manejo de errores
   - No hay nodos de error handling
   - Recomendación: Agregar manejo de errores para Redis y OpenAI

3. ⚠️ **CONFIGURACIÓN**: Requiere credenciales:
   - OpenAI API Key
   - Redis connection
   - WhatsApp API credentials

### Recomendaciones:
1. Agregar nodos de manejo de errores
2. Configurar timeout en HTTP requests
3. Validar formato de entrada de WhatsApp
4. Agregar logging para debugging

---

## Workflow 2: Agente de Productos
**Archivo:** `workflow-2-agente-productos.json`

### Estado: ❌ INVÁLIDO - REQUIERE CORRECCIONES

### Análisis Estructural:
- ✅ Estructura JSON correcta
- ✅ Campos requeridos presentes
- ✅ 14 nodos definidos

### Nodos Analizados:
1. **Webhook Productos** - ✅ Correcto
2. **Chat Model - OpenAI** - ✅ Correcto (temp: 0.5, max tokens: 500)
3. **Window Buffer Memory** - ✅ Correcto
4-6. **Tools: Buscar Producto, Consultar Stock, Obtener Precio**
   - ❌ **PROBLEMA CRÍTICO**: `workflowId: "={{ $parameter.workflowId }}"`
   - Este parámetro es inválido y causará error
   - **SOLUCIÓN**: Necesitas crear workflows separados para cada tool o usar HTTP Request directamente

7. **AI Agent Productos** - ✅ Prompt bien estructurado
8. **¿Necesita Escalamiento?** - ✅ Lógica correcta
9-10. **Envío WhatsApp y Escalamiento** - ✅ Correcto

11-13. **API Calls** (Buscar, Stock, Precio)
   - ⚠️ **PROBLEMA**: Estos nodos NO están conectados
   - Están diseñados como "subworkflows" pero no están integrados
   - **SOLUCIÓN**: Necesitan ser workflows separados O convertir Tools en HTTP Request tools

### Problemas CRÍTICOS Detectados:
1. ❌ **FATAL**: Las herramientas AI (Tool: Buscar Producto, etc.) usan `toolWorkflow` pero no tienen workflows asociados válidos
2. ❌ **FATAL**: Los nodos API (líneas 219-266) están desconectados del flujo principal
3. ⚠️ **ERROR DE DISEÑO**: Las herramientas deberían usar `@n8n/n8n-nodes-langchain.toolHttpRequest` en lugar de `toolWorkflow`

### Flujo de Datos Incorrecto:
```
Webhook → AI Agent → ¿Escalamiento? → WhatsApp/Escalamiento → Respond
                ↑
           [Tools NO CONECTADOS]
           [APIs NO CONECTADOS]
```

### CORRECCIÓN REQUERIDA:
Las herramientas deben ser rediseñadas para usar HTTP Request Tools:
```json
{
  "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
  "parameters": {
    "name": "buscar_producto",
    "description": "...",
    "method": "POST",
    "url": "http://localhost:3000/api/products/search"
  }
}
```

---

## Workflow 3: Agente de Pedidos
**Archivo:** `workflow-3-agente-pedidos.json`

### Estado: ❌ INVÁLIDO - MISMOS PROBLEMAS QUE WORKFLOW 2

### Problemas Detectados:
1. ❌ **FATAL**: Tools con `workflowId` inválido (mismo problema)
2. ❌ **FATAL**: Nodos API desconectados (líneas 253-317)
3. ❌ **ERROR DE DISEÑO**: Herramientas mal configuradas

### Herramientas Afectadas:
- Tool: Estado Pedido
- Tool: Rastrear Envío
- Tool: Buscar Pedidos
- Tool: Iniciar Devolución

### Misma corrección requerida que Workflow 2

---

## Workflow 4: Escalamiento a Soporte Humano
**Archivo:** `workflow-4-escalamiento-soporte.json`

### Estado: ✅ VÁLIDO CON OBSERVACIONES MENORES

### Análisis Estructural:
- ✅ Estructura JSON correcta
- ✅ 13 nodos bien conectados
- ✅ Flujo lógico correcto

### Nodos Analizados:
1. **Webhook Escalamiento** - ✅ Correcto
2. **Crear Datos de Ticket** - ✅ Genera ticketId único
3. **Guardar Ticket en BD** (PostgreSQL)
   - ✅ Query INSERT correcta
   - ✅ Parámetros bien mapeados
   - ⚠️ ADVERTENCIA: Requiere tabla `support_tickets` creada

4-6. **Notificaciones Paralelas**
   - ✅ Slack notification correcta
   - ✅ Email HTML bien formateado
   - ✅ WhatsApp response al cliente
   - ✅ Ejecución en paralelo correcta

7. **¿Es Urgente?** - ✅ Condiciones bien definidas
8. **Marcar como Urgente** - ✅ Query UPDATE correcta
9. **Alerta Urgente Slack** - ✅ Canal separado para urgentes
10. **Registrar Log** - ✅ Logging correcto

### Observaciones:
1. ⚠️ **CONFIGURACIÓN**: Requiere:
   - PostgreSQL con tablas creadas
   - Slack OAuth2
   - Email SMTP
   - Variables de entorno configuradas

2. ⚠️ **MEJORA**: El nodo "Alerta Urgente Slack" usa `authentication: "oAuth2"` pero el otro Slack no lo especifica
   - Inconsistencia en autenticación

3. ✅ **POSITIVO**: Buen manejo de casos urgentes con priorización

---

## Resumen General

### Workflows Válidos:
- ✅ Workflow 1: Clasificación de Mensajes (con observaciones menores)
- ✅ Workflow 4: Escalamiento a Soporte Humano (con observaciones menores)

### Workflows que REQUIEREN CORRECCIÓN:
- ❌ Workflow 2: Agente de Productos (CRÍTICO)
- ❌ Workflow 3: Agente de Pedidos (CRÍTICO)

---

## Problemas CRÍTICOS que Impiden Funcionamiento

### 1. Herramientas AI mal configuradas (Workflows 2 y 3)
**Problema:**
```json
"workflowId": "={{ $parameter.workflowId }}"
```
Este parámetro no existe y causará error.

**Solución Opción A - HTTP Request Tools:**
Reemplazar `toolWorkflow` por `toolHttpRequest`:
```json
{
  "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
  "parameters": {
    "name": "buscar_producto",
    "description": "Busca productos en el catálogo",
    "method": "POST",
    "url": "={{ $env.PRODUCTS_API_URL }}/search",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ { \"query\": $input.query } }}"
  }
}
```

**Solución Opción B - Code Tools:**
Usar `toolCode` para lógica personalizada:
```json
{
  "type": "@n8n/n8n-nodes-langchain.toolCode",
  "parameters": {
    "name": "buscar_producto",
    "description": "Busca productos",
    "code": "// Implementar búsqueda aquí"
  }
}
```

### 2. Nodos API desconectados
Los nodos de API (líneas 219-266 en Workflow 2) están en el JSON pero NO conectados al flujo.
- Deben eliminarse O
- Deben usarse como parte de las tools

### 3. Falta de manejo de errores
Ningún workflow tiene error handling configurado.

---

## Configuraciones Requeridas para Producción

### Variables de Entorno Necesarias:
```bash
# APIs
PRODUCTS_API_URL=https://api.empresa.com
ORDERS_API_URL=https://api.empresa.com

# Slack
SLACK_CHANNEL_ID=C12345678
SLACK_URGENT_CHANNEL_ID=C87654321

# Email
SUPPORT_EMAIL=support@empresa.com
```

### Credenciales a Configurar en n8n:
1. OpenAI API (para todos los AI Agents)
2. Redis (para memoria de conversaciones)
3. PostgreSQL (para tickets)
4. WhatsApp Business API
5. Slack OAuth2
6. SMTP Email

### Tablas de Base de Datos Requeridas:

```sql
-- Tabla para tickets de soporte
CREATE TABLE support_tickets (
    ticket_id VARCHAR(255) PRIMARY KEY,
    phone_number VARCHAR(50),
    customer_message TEXT,
    context JSONB,
    reason TEXT,
    agent_response TEXT,
    timestamp TIMESTAMP,
    status VARCHAR(50),
    priority VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tabla para logs de escalamiento
CREATE TABLE escalation_log (
    id SERIAL PRIMARY KEY,
    ticket_id VARCHAR(255),
    action VARCHAR(100),
    timestamp TIMESTAMP,
    FOREIGN KEY (ticket_id) REFERENCES support_tickets(ticket_id)
);
```

---

## Recomendaciones Finales

### Prioridad ALTA:
1. ❌ **CORREGIR Workflows 2 y 3** - No funcionarán en estado actual
2. ⚠️ Agregar manejo de errores a todos los workflows
3. ⚠️ Configurar todas las credenciales antes de importar
4. ⚠️ Crear tablas de base de datos

### Prioridad MEDIA:
1. Cambiar URLs localhost por URLs de producción
2. Agregar validación de formato de mensajes WhatsApp
3. Implementar rate limiting en webhooks
4. Agregar logging más detallado

### Prioridad BAJA:
1. Optimizar prompts de AI
2. Ajustar parámetros de temperatura según casos de uso
3. Implementar métricas y analytics
4. Agregar tests automatizados

---

## Próximos Pasos

1. **Inmediato**: Corregir Workflows 2 y 3 usando HTTP Request Tools
2. **Antes de importar**: Crear tablas de base de datos y configurar credenciales
3. **Al importar**: Verificar todas las conexiones de nodos
4. **Post-importación**: Probar cada workflow individualmente con datos de prueba
5. **Antes de producción**: Implementar manejo de errores completo

---

## Evaluación Final

| Workflow | Estado | Funcionalidad | Requiere Acción |
|----------|--------|---------------|-----------------|
| 1. Clasificación | ✅ VÁLIDO | 85% | Configuración |
| 2. Productos | ❌ INVÁLIDO | 0% | **CORRECCIÓN CRÍTICA** |
| 3. Pedidos | ❌ INVÁLIDO | 0% | **CORRECCIÓN CRÍTICA** |
| 4. Escalamiento | ✅ VÁLIDO | 90% | Configuración |

**Conclusión:** El sistema tiene una arquitectura bien diseñada, pero los workflows 2 y 3 necesitan corrección inmediata en las herramientas AI antes de poder ser utilizados.
