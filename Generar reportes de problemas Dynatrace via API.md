## 1. Preparación inicial

Primero necesitas tener:

- **URL de tu entorno Dynatrace** (ej: `https://your_environment.live.dynatrace.com`)
    
- **Token de API** con permisos de lectura para problemas
    

## 2. Estructura básica del script

Te recomiendo usar Python para esto. Aquí tienes un script base:

```
import requests
import pandas as pd
from datetime import datetime, timedelta
import json

# Configuración
DYNATRACE_URL = "https://your_environment.live.dynatrace.com"
API_TOKEN = "tu_token_aqui"
MANAGED_ZONE = "Segurdok"

def get_problems_from_dynatrace(start_time, end_time):
    headers = {
        "Authorization": f"Api-Token {API_TOKEN}",
        "Content-Type": "application/json"
    }
    
    # Parámetros para filtrar por fecha
    params = {
        "from": start_time,
        "to": end_time,
        "managedZone": MANAGED_ZONE
    }
    
    endpoint = f"{DYNATRACE_URL}/api/v2/problems"
    
    try:
        response = requests.get(endpoint, headers=headers, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error calling API: {e}")
        return None

def problems_to_csv(problems_data, filename):
    if not problems_data or 'problems' not in problems_data:
        print("No problems data found")
        return
    
    problems_list = []
    
    for problem in problems_data['problems']:
        problem_info = {
            'problem_id': problem.get('problemId', ''),
            'title': problem.get('title', ''),
            'status': problem.get('status', ''),
            'severity_level': problem.get('severityLevel', ''),
            'impact_level': problem.get('impactLevel', ''),
            'affected_entities': ', '.join([entity.get('name', '') for entity in problem.get('affectedEntities', [])]),
            'start_time': problem.get('startTime', ''),
            'end_time': problem.get('endTime', ''),
            'management_zone': MANAGED_ZONE
        }
        problems_list.append(problem_info)
    
    df = pd.DataFrame(problems_list)
    df.to_csv(filename, index=False, encoding='utf-8')
    print(f"File saved: {filename} with {len(problems_list)} problems")

# Ejemplo para el mes actual
def get_current_month_problems():
    today = datetime.now()
    first_day = today.replace(day=1, hour=0, minute=0, second=0, microsecond=0)
    
    if today.month == 12:
        next_month = today.replace(year=today.year + 1, month=1, day=1)
    else:
        next_month = today.replace(month=today.month + 1, day=1)
    
    last_day = next_month - timedelta(days=1)
    
    start_time = first_day.isoformat() + "Z"
    end_time = last_day.isoformat() + "Z"
    
    print(f"Fetching problems from {start_time} to {end_time}")
    
    problems_data = get_problems_from_dynatrace(start_time, end_time)
    
    if problems_data:
        filename = f"problems_{MANAGED_ZONE}_{today.strftime('%Y-%m')}.csv"
        problems_to_csv(problems_data, filename)

if __name__ == "__main__":
    get_current_month_problems()
```

## 3. Script más avanzado con filtros adicionales

```
import requests
import pandas as pd
from datetime import datetime, timedelta
import argparse

class DynatraceProblemExporter:
    def __init__(self, base_url, api_token):
        self.base_url = base_url.rstrip('/')
        self.api_token = api_token
        self.headers = {
            "Authorization": f"Api-Token {api_token}",
            "Content-Type": "application/json"
        }
    
    def get_problems(self, start_time, end_time, managed_zone=None, status=None):
        params = {
            "from": start_time,
            "to": end_time
        }
        
        if managed_zone:
            params["managedZone"] = managed_zone
        
        if status:
            params["status"] = status
            
        endpoint = f"{self.base_url}/api/v2/problems"
        
        all_problems = []
        next_page_key = None
        
        try:
            while True:
                if next_page_key:
                    params['nextPageKey'] = next_page_key
                
                response = requests.get(endpoint, headers=self.headers, params=params)
                response.raise_for_status()
                
                data = response.json()
                all_problems.extend(data.get('problems', []))
                
                next_page_key = data.get('nextPageKey')
                if not next_page_key:
                    break
                    
            return {'problems': all_problems}
            
        except requests.exceptions.RequestException as e:
            print(f"Error calling API: {e}")
            return None
    
    def export_to_csv(self, problems_data, filename):
        if not problems_data or 'problems' not in problems_data:
            print("No problems data found")
            return False
        
        problems_list = []
        
        for problem in problems_data['problems']:
            # Procesar información básica
            problem_info = {
                'problem_id': problem.get('problemId', ''),
                'title': problem.get('title', ''),
                'status': problem.get('status', ''),
                'severity_level': problem.get('severityLevel', ''),
                'impact_level': problem.get('impactLevel', ''),
                'priority_level': problem.get('priorityLevel', ''),
                'start_time': self.format_timestamp(problem.get('startTime')),
                'end_time': self.format_timestamp(problem.get('endTime')),
                'root_cause_entity': problem.get('rootCauseEntity', {}).get('name', ''),
                'affected_entities_count': len(problem.get('affectedEntities', [])),
                'management_zones': ', '.join([mz.get('name', '') for mz in problem.get('managementZones', [])])
            }
            
            problems_list.append(problem_info)
        
        df = pd.DataFrame(problems_list)
        df.to_csv(filename, index=False, encoding='utf-8')
        print(f"✅ File saved: {filename} with {len(problems_list)} problems")
        return True
    
    def format_timestamp(self, timestamp):
        if not timestamp:
            return ""
        try:
            # Convertir timestamp a formato legible
            dt = datetime.fromtimestamp(timestamp / 1000)
            return dt.strftime('%Y-%m-%d %H:%M:%S')
        except:
            return timestamp

def main():
    parser = argparse.ArgumentParser(description='Export Dynatrace problems to CSV')
    parser.add_argument('--url', required=True, help='Dynatrace environment URL')
    parser.add_argument('--token', required=True, help='API Token')
    parser.add_argument('--managed-zone', default='Segurdok', help='Managed zone name')
    parser.add_argument('--month', help='Month in format YYYY-MM (e.g., 2024-01)')
    parser.add_argument('--output', help='Output filename')
    
    args = parser.parse_args()
    
    # Calcular fechas
    if args.month:
        year, month = map(int, args.month.split('-'))
        start_date = datetime(year, month, 1)
        if month == 12:
            end_date = datetime(year + 1, 1, 1) - timedelta(days=1)
        else:
            end_date = datetime(year, month + 1, 1) - timedelta(days=1)
    else:
        # Mes actual por defecto
        today = datetime.now()
        start_date = today.replace(day=1)
        end_date = today
    
    end_date = end_date.replace(hour=23, minute=59, second=59)
    
    start_time = start_date.isoformat() + "Z"
    end_time = end_date.isoformat() + "Z"
    
    # Generar nombre de archivo
    if not args.output:
        month_str = start_date.strftime('%Y-%m')
        args.output = f"problems_{args.managed_zone}_{month_str}.csv"
    
    # Ejecutar exportación
    exporter = DynatraceProblemExporter(args.url, args.token)
    
    print(f"📊 Fetching problems for {args.managed_zone}")
    print(f"📅 Period: {start_date.strftime('%Y-%m-%d')} to {end_date.strftime('%Y-%m-%d')}")
    
    problems_data = exporter.get_problems(start_time, end_time, args.managed_zone)
    
    if problems_data:
        exporter.export_to_csv(problems_data, args.output)

if __name__ == "__main__":
    main()
```

## 4. Cómo usar el script

### Instalar dependencias:

```
pip install requests pandas
```

### Ejecutar:

```
# Para el mes actual
python dynatrace_exporter.py --url "https://your_env.live.dynatrace.com" --token "dt0c01.XXX" --managed-zone "Segurdok"

# Para un mes específico
python dynatrace_exporter.py --url "https://your_env.live.dynatrace.com" --token "dt0c01.XXX" --managed-zone "Segurdok" --month "2024-01"

# Con archivo de salida personalizado
python dynatrace_exporter.py --url "https://your_env.live.dynatrace.com" --token "dt0c01.XXX" --managed-zone "Segurdok" --output "mis_problemas.csv"
```

## 5. Configuración del Token API

En Dynatrace:

1. Ve a **Settings** → **Integration** → **Dynatrace API**
    
2. Crea un nuevo token con los permisos:
    
    - `problems.read`
        
    - `DataExport` (opcional)
        

## 6. Campos que incluye el CSV

- `problem_id`: ID único del problema
    
- `title`: Título del problema
    
- `status`: Estado (OPEN, CLOSED, etc.)
    
- `severity_level`: Nivel de severidad
    
- `impact_level`: Nivel de impacto
    
- `start_time/end_time`: Fechas de inicio/fin
    
- `root_cause_entity`: Entidad causante
    
- `affected_entities_count`: Número de entidades afectadas
    
- `management_zones`: Zonas de gestión

e explico paso a paso cómo crear el token API con los permisos necesarios:

## 1. Acceder a la configuración de tokens API

**Método 1:**

- Ve a **Settings** (engranaje abajo a la izquierda)
    
- Selecciona **Integration**
    
- Elige **Dynatrace API**
    

**Método 2:**

- Navega directamente a: `https://{tu-entorno}.live.dynatrace.com/#settings/tokens`
    

## 2. Crear nuevo token

1. Haz clic en el botón **"Generate new token"**
    
2. Completa la información:
    
    - **Token name**: `Problemas-Segurdok-Report` (o el nombre que prefieras)
        
    - **Expiration date**: Recomiendo seleccionar una fecha larga (ej: 1 año) o "No expiration"
        
    - **Scope**: Aquí es donde defines los permisos
        

## 3. Configurar los permisos (Scope)

En la sección **Scope**, busca y selecciona estos permisos:

### Permisos principales necesarios:

- **`problems.read`** - Para leer la información de problemas
    
- **`DataExport`** - Para exportar datos (opcional pero recomendado)
    

### Cómo seleccionarlos:

1. En el buscador de scopes, escribe **"problems"**
    
2. Marca la casilla de **`problems.read`**
    
3. Busca **"DataExport"** y márcala también
    
4. Puedes añadir **`entities.read`** si quieres información más detallada de las entidades
    

## 4. Guardar el token

1. Haz clic en **"Generate"**
    
2. **⚠️ IMPORTANTE: Copia el token inmediatamente** - Solo se muestra una vez
    
3. Guárdalo en un lugar seguro (gestor de contraseñas, variable de entorno, etc.)
    

## 5. Verificación visual

Cuando crees el token, debería verse así en la configuración:

```
Token name: Problemas-Segurdok-Report
Scopes: problems.read, DataExport
Status: Active
```

## 6. Alternativa: Crear el token via API (avanzado)

Si prefieres crear el token programáticamente:

```
curl -X POST "https://{your-environment}/api/v2/apiTokens" \
  -H "Content-Type: application/json" \
  -H "Authorization: Api-Token {tu-token-existente}" \
  -d '{
    "name": "Problemas-Segurdok-Report",
    "scopes": [
      "problems.read",
      "DataExport"
    ],
    "expirationDate": "2025-12-31"
  }'
```

## 7. Uso en el script

Una vez tengas el token, actualiza el script:

```
API_TOKEN = "dt0c01.TU_TOKEN_AQUI.XXX..."
```

## 8. Verificación de permisos

Puedes verificar que el token funciona con este comando:

```
curl -H "Authorization: Api-Token {tu-token}" \
  "https://{tu-entorno}/api/v2/problems?from=now-1h"
```

## Recomendaciones de seguridad:

1. **No compartas** el token
    
2. **Usa variables de entorno** en lugar de hardcodearlo:

```
import os
API_TOKEN = os.getenv('DYNATRACE_TOKEN')
```

1. **Restringe** el token solo a los permisos necesarios
    
2. **Revoca** el token si ya no lo usas