# Reglas MCP - Implementación de Servidores MCP en Rails

---

## REGLAS CRÍTICAS - LEER ANTES DE IMPLEMENTAR

Estas 4 reglas son **OBLIGATORIAS** para que el servidor MCP funcione. El incumplimiento causará fallos silenciosos.

---

## 1. CONFIGURACIÓN DEL SERVIDOR (lib/mcp/server.rb)

### CORRECTO
````ruby
module Mcp
  class Server
    def initialize
      @server = MCP::Server.new(
        name: "proyecto-mcp-server",
        version: "0.1.0",
        tools: [
          # Pasar CLASES directamente
          Mcp::Tools::Companies::ListCompaniesTool,
          Mcp::Tools::Companies::GetCompanyTool,
          Mcp::Tools::Contacts::ListContactsTool,
          # ... todas las herramientas
        ]
      )
    end

    def start
      # Sin logs en stdout
      transport = MCP::Server::Transports::StdioTransport.new(@server)
      transport.open
    rescue => e
      # Logs de error a stderr
      $stderr.puts "[MCP] Error: #{e.message}"
      raise
    end
  end
end
````

### NUNCA HACER
````ruby
# NO usar define_tool con bloques
def initialize
  @server = MCP::Server.new(name: "server", version: "0.1.0", tools: [])
  setup_tools  # NO HACER ESTO
end

def setup_tools
  @server.define_tool(name: "list_companies", ...) do |args, context|
    # Este bloque NUNCA se ejecutará correctamente
    Mcp::Tools::Companies::ListCompaniesTool.call(...)
  end
end

# NO poner logs en stdout
def start
  puts "Starting server..."              # Rompe JSON-RPC
  Rails.logger.info "Listening..."       # Rompe JSON-RPC
  transport = MCP::Server::Transports::StdioTransport.new(@server)
  transport.open
end
````

---

## 2. FORMATO DE RESPUESTA (Todas las Tools)

### CORRECTO
````ruby
module Mcp
  module Tools
    module Companies
      class ListCompaniesTool < MCP::Tool
        description "List all companies"
        
        input_schema(
          type: "object",
          properties: {
            limit: { type: "integer", default: 100 },
            offset: { type: "integer", default: 0 }
          }
        )

        def self.call(limit: 100, offset: 0, server_context:)
          companies = CompanyQueryBuilder.list(limit: limit, offset: offset)
          data = CompanySerializer.serialize_collection(companies)
          
          result = {
            companies: data,
            total: companies.size,
            limit: limit,
            offset: offset
          }

          # SIEMPRE devolver MCP::Tool::Response
          MCP::Tool::Response.new([
            { type: "text", text: JSON.pretty_generate(result) }
          ])
        end
      end
    end
  end
end
````

### NUNCA DEVOLVER
````ruby
def self.call(limit: 100, offset: 0, server_context:)
  result = { companies: data, total: total }
  
  # NINGUNA de estas funciona:
  result                                           
  [{ type: "text", text: result.to_json }]        
  { content: [{ type: "text", text: ... }] }      
  JSON.pretty_generate(result)                    
  { companies: data }                             
end
````

---

## 3. NO LOGS EN STDOUT

### Regla de Oro

**TODO lo que vaya a `stdout` debe ser JSON-RPC válido. Cualquier otra cosa rompe el protocolo.**

### Permitido
````ruby
# Logs a stderr
$stderr.puts "[MCP] Debug info"

# Sin logs en producción
# (silencio total en stdout)
````

### Prohibido
````ruby
# Cualquier puts
puts "Starting..."
puts "Processing..."

# Rails.logger (va a stdout en stdio transport)
Rails.logger.info "..."
Rails.logger.debug "..."

# Emojis o texto decorativo
puts "Starting..."
puts "Ready!"
````

**Síntoma si se rompe:** Claude Desktop mostrará errores como:
````
Unexpected token ' ', "Startin"... is not valid JSON
````

---

## 4. CONFIGURACIÓN CLAUDE DESKTOP

### PASO 1: Crear Script de Inicio

**Windows - `start_mcp.bat`:**
````batch
@echo off
cd /d "%~dp0"
bundle exec rails mcp:server
````

**macOS/Linux - `start_mcp.sh`:**
````bash
#!/bin/bash
cd "$(dirname "$0")"
bundle exec rails mcp:server
````
````bash
chmod +x start_mcp.sh
````

### PASO 2: Configurar Claude Desktop

**Ubicación del archivo:**
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

**Contenido:**
````json
{
  "mcpServers": {
    "NOMBRE_PROYECTO-server": {
      "command": "RUTA_ABSOLUTA_AL_PROYECTO/start_mcp.bat",
      "args": [],
      "env": {
        "RAILS_ENV": "development"
      }
    }
  }
}
````

**Ejemplo Windows:**
````json
{
  "mcpServers": {
    "NOMBRE_PROYECTO-server": {
      "command": "RUTA_ABSOLUTA_AL_PROYECTO/start_mcp.bat",
      "args": [],
      "env": {
        "RAILS_ENV": "development"
      }
    }
  }
}
````

**Ejemplo macOS/Linux:**
````json
{
  "mcpServers": {
    "NOMBRE_PROYECTO-server": {
      "command": "RUTA_ABSOLUTA_AL_PROYECTO/start_mcp.sh",
      "args": [],
      "env": {
        "RAILS_ENV": "development"
      }
    }
  }
}
````

### IMPORTANTE

1. **Usar RUTA ABSOLUTA** al script
2. **Windows**: Usar `\\` o `/` (NO `\`)
3. **Reiniciar Claude Desktop** completamente después de cambios
4. Verificar en Settings → Developer → MCP que aparezca conectado

**Nota:** En los ejemplos anteriores, reemplazar:
- `NOMBRE_PROYECTO` con el nombre real del proyecto
- `RUTA_ABSOLUTA_AL_PROYECTO` con la ruta completa donde se encuentra el archivo `.claude` de este proyecto

---

## Estructura de Archivos Requerida
````
proyecto/
├── start_mcp.bat / start_mcp.sh    # Script de inicio
├── lib/
│   ├── mcp/
│   │   ├── server.rb               # Servidor (Regla #1)
│   │   ├── logger.rb               # Logger opcional
│   │   ├── tools/                  # Todas las herramientas (Regla #2)
│   │   │   ├── companies/
│   │   │   │   ├── list_companies_tool.rb
│   │   │   │   ├── get_company_tool.rb
│   │   │   │   └── search_companies_tool.rb
│   │   │   ├── contacts/
│   │   │   ├── deals/
│   │   │   └── ...
│   │   ├── serializers/            # Serialización de datos
│   │   │   ├── company_serializer.rb
│   │   │   └── ...
│   │   └── query_builders/         # Queries a BD
│   │       ├── company_query_builder.rb
│   │       └── ...
│   └── tasks/
│       └── mcp.rake                # Rake task
├── config/
│   └── claude_desktop_config.json.example  # Template de config
└── db/
    └── seeds.rb                    # Datos de prueba
````

---

## Checklist de Implementación

### Fase 1: Setup
- Añadir `gem "mcp"` al Gemfile
- Ejecutar `bundle install`
- Crear estructura de carpetas `lib/mcp/`
- Crear `lib/tasks/mcp.rake`

### Fase 2: Servidor (REGLA #1)
- Crear `lib/mcp/server.rb`
- Pasar herramientas en array `tools: []`
- NO usar `define_tool` con bloques
- Método `start` sin logs en stdout

### Fase 3: Herramientas (REGLA #2)
- Cada tool hereda de `MCP::Tool`
- Incluir `description`
- Incluir `input_schema` completo
- TODAS devuelven `MCP::Tool::Response.new([...])`

### Fase 4: Scripts (REGLA #4)
- Crear `start_mcp.bat` (Windows) o `start_mcp.sh` (Unix)
- Script incluye `cd` al directorio del proyecto
- Permisos de ejecución en Unix: `chmod +x start_mcp.sh`

### Fase 5: Configuración
- Crear `config/claude_desktop_config.json.example`
- Documentar rutas en README.md
- Generar script de ayuda `setup_claude_desktop.rb` (opcional)

### Fase 6: Testing
- Crear seeds de prueba en `db/seeds.rb`
- Verificar en consola: `rails console`
- Probar herramienta directamente: `Tool.call(...)`
- Ejecutar servidor: `bundle exec rails mcp:server`

### Fase 7: Integración Claude Desktop
- Copiar configuración a archivo de Claude Desktop
- Reiniciar Claude Desktop
- Verificar conexión en Settings → Developer → MCP
- Probar consulta: "¿Cuántas empresas tengo?"

---

## Errores Comunes y Soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| "Unexpected token in JSON" | Logs en stdout (Regla #3) | Eliminar todos los `puts`/`Rails.logger` |
| "Internal error calling tool" | Formato incorrecto (Regla #2) | Usar `MCP::Tool::Response.new([...])` |
| "Tool execution failed" | Bloques `define_tool` (Regla #1) | Pasar clases en array `tools: []` |
| "Could not locate Gemfile" | Script sin `cd` (Regla #4) | Añadir `cd` al directorio en script |
| Server no aparece en MCP | Ruta incorrecta (Regla #4) | Usar ruta absoluta en JSON |
| "No Rakefile found" | No usar bundle exec | `bundle exec rails mcp:server` |

---

## Comandos de Verificación

### Verificar datos en BD
````bash
rails console
````
````ruby
Company.count
Company.first
````

### Probar herramienta directamente
````ruby
result = Mcp::Tools::Companies::ListCompaniesTool.call(
  limit: 1,
  offset: 0,
  server_context: nil
)

puts result.inspect
# Debe mostrar: #<MCP::Tool::Response:...>
````

### Ejecutar servidor manualmente
````bash
cd /ruta/al/proyecto
bundle exec rails mcp:server
````

Debe quedarse esperando **sin mostrar mensajes**. Si aparece texto, rompe la Regla #3.

### Ver logs de Claude Desktop
- Windows: `%APPDATA%\Claude\logs\`
- macOS: `~/Library/Logs/Claude/`
- O desde Claude Desktop: Settings → Developer → View Logs

---

## Template de README.md para el Proyecto

Incluir esta sección en el README del proyecto:
````markdown
## Configuración de Servidor MCP

Este proyecto incluye un servidor MCP para integración con Claude Desktop.

### Setup Rápido

1. Instalar dependencias:
```bash
bundle install
rails db:migrate
rails db:seed
```

2. Crear script de inicio:
   - Windows: Ya existe `start_mcp.bat`
   - Unix: Ya existe `start_mcp.sh` (ejecutar `chmod +x start_mcp.sh`)

3. Configurar Claude Desktop:
   - Abrir: `%APPDATA%\Claude\claude_desktop_config.json` (Windows)
   - Añadir configuración del archivo `config/claude_desktop_config.json.example`
   - Reemplazar `RUTA_PROYECTO` con la ruta absoluta a este proyecto

4. Reiniciar Claude Desktop completamente

5. Verificar:
   - Settings → Developer → MCP
   - Debe aparecer "proyecto-server" conectado

### Troubleshooting

Ver archivo `.claude/templates/mcp-rules.md` para reglas detalladas y solución de problemas.
````

---

## Resumen de las 4 Reglas Críticas

1. **Servidor**: Pasar clases en `tools: []`, NO `define_tool`
2. **Respuestas**: SIEMPRE `MCP::Tool::Response.new([...])`
3. **Logs**: NUNCA `puts`/`Rails.logger` en stdout
4. **Configuración**: Script con `cd` + ruta absoluta en JSON

**Seguir estas 4 reglas garantiza que el servidor MCP funcione correctamente desde el primer intento.**
