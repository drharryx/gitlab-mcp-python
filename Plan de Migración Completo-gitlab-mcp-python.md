# Migración de GitLab MCP Server de TypeScript a Python

## Resumen Ejecutivo

Este documento describe el plan completo para migrar el proyecto [gitlab-mcp](https://github.com/zereight/gitlab-mcp) de TypeScript/Node.js a Python, manteniendo toda la funcionalidad existente y aprovechando el ecosistema Python para MCP servers.

## Análisis del Proyecto Original

### Características Principales
- **83 herramientas GitLab API** organizadas por categorías
- **Múltiples transportes**: STDIO, SSE, Streamable HTTP  
- **Autenticación flexible**: Tokens, cookies, proxies
- **Configuración completa** de variables de entorno
- **Validación robusta** con esquemas Zod

### Arquitectura Actual (TypeScript)
```
gitlab-mcp/
├── index.ts              # Servidor MCP principal
├── schemas.ts            # Esquemas Zod para validación
├── customSchemas.ts      # Esquemas personalizados
├── package.json          # Dependencias Node.js
├── tsconfig.json         # Configuración TypeScript
└── test/                 # Tests y utilidades
```

## Plan de Migración a Python

### Tecnologías Seleccionadas

#### Framework Principal
- **FastMCP**: SDK Python para servidores MCP, similar a FastAPI
- **Ventajas**: Sintaxis simple, decoradores, validación automática

#### Dependencias Core
- **python-gitlab**: Cliente oficial GitLab API para Python
- **pydantic**: Validación de datos y serialización
- **httpx**: Cliente HTTP asíncrono moderno
- **python-dotenv**: Manejo de variables de entorno

### Nueva Arquitectura (Python)

```
gitlab-mcp-python/
├── main.py               # Servidor MCP principal
├── gitlab_client.py      # Wrapper cliente GitLab
├── schemas.py            # Esquemas Pydantic
├── config.py             # Configuración y variables
├── tools/                # Herramientas organizadas
│   ├── __init__.py
│   ├── repository.py     # Gestión repositorios
│   ├── issues.py         # Gestión issues
│   ├── merge_requests.py # Gestión MRs
│   ├── pipelines.py      # Gestión pipelines
│   ├── wiki.py           # Gestión wiki
│   └── milestones.py     # Gestión milestones
├── requirements.txt      # Dependencias Python
├── pyproject.toml        # Configuración proyecto
├── README.md             # Documentación
└── tests/                # Tests unitarios
```

## Implementación Paso a Paso

### Paso 1: Configuración del Entorno

```bash
# Crear proyecto Python
mkdir gitlab-mcp-python
cd gitlab-mcp-python

# Configurar entorno virtual
python -m venv venv
source venv/bin/activate  # En Windows: venv\Scripts\activate

# Instalar dependencias
pip install fastmcp python-gitlab pydantic httpx python-dotenv
```

### Paso 2: Archivo de Dependencias

**requirements.txt**
```txt
fastmcp>=0.1.0
python-gitlab>=4.0.0
pydantic>=2.0.0
httpx>=0.25.0
python-dotenv>=1.0.0
typing-extensions>=4.5.0
```

### Paso 3: Configuración Base

**config.py**
```python
import os
from typing import Optional, List
from pydantic import BaseSettings, Field

class GitLabConfig(BaseSettings):
    personal_access_token: str = Field(..., env="GITLAB_PERSONAL_ACCESS_TOKEN")
    api_url: str = Field("https://gitlab.com/api/v4", env="GITLAB_API_URL") 
    project_id: Optional[str] = Field(None, env="GITLAB_PROJECT_ID")
    allowed_project_ids: Optional[str] = Field(None, env="GITLAB_ALLOWED_PROJECT_IDS")
    read_only_mode: bool = Field(False, env="GITLAB_READ_ONLY_MODE")
    use_gitlab_wiki: bool = Field(False, env="USE_GITLAB_WIKI")
    use_milestone: bool = Field(False, env="USE_MILESTONE") 
    use_pipeline: bool = Field(False, env="USE_PIPELINE")
    
    class Config:
        env_file = ".env"

# Instancia global de configuración
config = GitLabConfig()
```

### Paso 4: Cliente GitLab Wrapper

**gitlab_client.py**
```python
import gitlab
from typing import Optional, Dict, Any, List
from config import config

class GitLabClient:
    def __init__(self):
        self.gl = gitlab.Gitlab(
            url=config.api_url,
            private_token=config.personal_access_token
        )
        
    def get_project(self, project_id: str):
        """Obtiene un proyecto por ID"""
        return self.gl.projects.get(project_id)
        
    def search_projects(self, search: str, **kwargs):
        """Busca proyectos"""
        return self.gl.projects.list(search=search, **kwargs)
        
    def get_effective_project_id(self, project_id: Optional[str] = None) -> str:
        """Obtiene el ID efectivo del proyecto"""
        if config.project_id:
            return config.project_id
        if project_id:
            return project_id
        raise ValueError("No project ID specified")

# Instancia global del cliente
gitlab_client = GitLabClient()
```

### Paso 5: Esquemas Pydantic

**schemas.py**
```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any, Union
from datetime import datetime

class ProjectSchema(BaseModel):
    project_id: str = Field(..., description="Project ID or URL-encoded path")

class CreateIssueSchema(ProjectSchema):
    title: str = Field(..., description="Issue title")
    description: Optional[str] = Field(None, description="Issue description")
    assignee_ids: Optional[List[int]] = Field(None, description="Array of user IDs to assign")
    labels: Optional[List[str]] = Field(None, description="Array of label names")
    milestone_id: Optional[str] = Field(None, description="Milestone ID to assign")

class SearchRepositoriesSchema(BaseModel):
    search: str = Field(..., description="Search query")
    page: Optional[int] = Field(1, description="Page number")
    per_page: Optional[int] = Field(20, description="Items per page")

class FileContentSchema(ProjectSchema):
    file_path: str = Field(..., description="Path to the file")
    ref: Optional[str] = Field(None, description="Branch/tag/commit reference")

# ... más esquemas según necesidades
```

### Paso 6: Servidor MCP Principal

**main.py**
```python
import asyncio
from fastmcp import FastMCP
from typing import Dict, Any, List, Optional

from config import config
from gitlab_client import gitlab_client
from schemas import *

# Crear instancia del servidor MCP
mcp = FastMCP(
    name="gitlab-mcp-python",
    description="GitLab MCP Server implementado en Python"
)

@mcp.tool()
def search_repositories(params: SearchRepositoriesSchema) -> Dict[str, Any]:
    """Buscar repositorios GitLab"""
    try:
        projects = gitlab_client.search_projects(
            search=params.search,
            page=params.page, 
            per_page=params.per_page
        )
        return {
            "success": True,
            "projects": [proj.asdict() for proj in projects],
            "count": len(projects)
        }
    except Exception as e:
        return {"success": False, "error": str(e)}

@mcp.tool() 
def get_file_contents(params: FileContentSchema) -> Dict[str, Any]:
    """Obtener contenido de un archivo"""
    try:
        project = gitlab_client.get_project(params.project_id)
        file = project.files.get(
            file_path=params.file_path,
            ref=params.ref or project.default_branch
        )
        
        return {
            "success": True,
            "file_name": file.file_name,
            "file_path": file.file_path,
            "content": file.decode().decode('utf-8'),
            "size": file.size,
            "ref": params.ref or project.default_branch
        }
    except Exception as e:
        return {"success": False, "error": str(e)}

@mcp.tool()
def create_issue(params: CreateIssueSchema) -> Dict[str, Any]:
    """Crear una nueva issue"""
    if config.read_only_mode:
        return {"success": False, "error": "Read-only mode enabled"}
        
    try:
        project = gitlab_client.get_project(params.project_id)
        issue_data = {
            'title': params.title,
            'description': params.description or '',
        }
        
        if params.assignee_ids:
            issue_data['assignee_ids'] = params.assignee_ids
        if params.labels:
            issue_data['labels'] = params.labels
        if params.milestone_id:
            issue_data['milestone_id'] = int(params.milestone_id)
            
        issue = project.issues.create(issue_data)
        
        return {
            "success": True,
            "issue": {
                "id": issue.id,
                "iid": issue.iid,
                "title": issue.title,
                "description": issue.description,
                "state": issue.state,
                "web_url": issue.web_url
            }
        }
    except Exception as e:
        return {"success": False, "error": str(e)}

# Herramientas adicionales se pueden cargar desde módulos separados
from tools.repository import register_repository_tools
from tools.issues import register_issue_tools
from tools.merge_requests import register_mr_tools

# Registrar herramientas por categorías
register_repository_tools(mcp, gitlab_client, config)
register_issue_tools(mcp, gitlab_client, config)
register_mr_tools(mcp, gitlab_client, config)

if __name__ == "__main__":
    # Ejecutar servidor MCP con transporte stdio
    mcp.run(transport="stdio")
```

### Paso 7: Herramientas Modularizadas

**tools/issues.py**
```python
from fastmcp import FastMCP
from typing import Dict, Any
from schemas import *

def register_issue_tools(mcp: FastMCP, gitlab_client, config):
    
    @mcp.tool()
    def list_issues(params: Dict[str, Any]) -> Dict[str, Any]:
        """Listar issues del proyecto"""
        try:
            project_id = params.get('project_id')
            project = gitlab_client.get_project(project_id)
            
            # Parámetros de filtrado
            state = params.get('state', 'opened')
            labels = params.get('labels', [])
            assignee_id = params.get('assignee_id')
            
            issues = project.issues.list(
                state=state,
                labels=labels,
                assignee_id=assignee_id,
                all=True
            )
            
            return {
                "success": True,
                "issues": [
                    {
                        "id": issue.id,
                        "iid": issue.iid, 
                        "title": issue.title,
                        "state": issue.state,
                        "labels": issue.labels,
                        "assignees": [a.name for a in issue.assignees] if issue.assignees else [],
                        "web_url": issue.web_url,
                        "created_at": issue.created_at,
                        "updated_at": issue.updated_at
                    }
                    for issue in issues
                ],
                "count": len(issues)
            }
        except Exception as e:
            return {"success": False, "error": str(e)}

    @mcp.tool()
    def update_issue(params: Dict[str, Any]) -> Dict[str, Any]:
        """Actualizar una issue existente"""
        if config.read_only_mode:
            return {"success": False, "error": "Read-only mode enabled"}
            
        try:
            project_id = params.get('project_id')
            issue_iid = params.get('issue_iid')
            
            project = gitlab_client.get_project(project_id)
            issue = project.issues.get(issue_iid)
            
            # Actualizar campos según parámetros
            if 'title' in params:
                issue.title = params['title']
            if 'description' in params:
                issue.description = params['description']
            if 'labels' in params:
                issue.labels = params['labels']
            if 'assignee_ids' in params:
                issue.assignee_ids = params['assignee_ids']
                
            issue.save()
            
            return {
                "success": True,
                "issue": {
                    "id": issue.id,
                    "iid": issue.iid,
                    "title": issue.title,
                    "state": issue.state,
                    "web_url": issue.web_url
                }
            }
        except Exception as e:
            return {"success": False, "error": str(e)}
```

### Paso 8: Variables de Entorno

**.env**
```bash
GITLAB_PERSONAL_ACCESS_TOKEN=your_gitlab_token_here
GITLAB_API_URL=https://gitlab.com/api/v4
GITLAB_PROJECT_ID=optional_default_project_id
GITLAB_READ_ONLY_MODE=false
USE_GITLAB_WIKI=true
USE_MILESTONE=true
USE_PIPELINE=true
```

### Paso 9: Testing

**tests/test_main.py**
```python
import pytest
from unittest.mock import Mock, patch
from main import mcp
from schemas import SearchRepositoriesSchema

def test_search_repositories():
    params = SearchRepositoriesSchema(
        search="test project",
        page=1,
        per_page=20
    )
    
    with patch('gitlab_client.gitlab_client') as mock_client:
        # Mock del resultado
        mock_projects = [Mock()]
        mock_projects[0].asdict.return_value = {"id": 1, "name": "Test Project"}
        mock_client.search_projects.return_value = mock_projects
        
        result = search_repositories(params)
        
        assert result["success"] == True
        assert len(result["projects"]) == 1
        assert result["projects"][0]["name"] == "Test Project"

# Más tests...
```

## Pasos Adicionales Requeridos

### 1. **Análisis detallado del código TypeScript**
- Revisar cada una de las 83 herramientas implementadas
- Identificar patrones comunes y lógica específica
- Mapear esquemas Zod a Pydantic equivalentes

### 2. **Implementación incremental**
- Comenzar con herramientas básicas (search, get_file_contents)
- Continuar con operaciones CRUD (create_issue, update_issue)
- Finalizar con operaciones avanzadas (pipelines, merge requests)

### 3. **Testing exhaustivo**
- Tests unitarios para cada herramienta
- Tests de integración con GitLab API
- Validación de compatibilidad con el servidor original

### 4. **Documentación completa**
- README detallado con instrucciones de instalación
- Ejemplos de uso para cada herramienta
- Guía de migración desde la versión TypeScript

### 5. **Configuración de deployment**
- Docker container para facilitar distribución
- Scripts de instalación automatizada
- Integración con Claude Desktop y otras herramientas MCP

## Cronograma Estimado

| Fase | Duración | Descripción |
|------|----------|-------------|
| **Fase 1** | 2-3 días | Setup inicial, configuración, herramientas básicas |
| **Fase 2** | 3-4 días | Migración de las 83 herramientas principales |
| **Fase 3** | 1-2 días | Testing, debugging y validación |
| **Fase 4** | 1 día | Documentación y empaquetado |

**Total estimado: 7-10 días de desarrollo**

## Ventajas de la Migración

### Para Desarrolladores Python
- **Ecosistema familiar**: Aprovechar herramientas Python existentes
- **FastMCP**: Framework más simple que la implementación TypeScript nativa
- **python-gitlab**: Cliente oficial con mejor mantenimiento

### Para el Proyecto
- **Menor superficie de ataque**: Dependencias más controladas
- **Mejor debugging**: Stack traces más claros en Python
- **Comunidad**: Mayor participación de desarrolladores Python

### Para Usuarios
- **Compatibilidad completa**: Mantiene todas las funcionalidades
- **Mejor rendimiento**: python-gitlab optimizado para operaciones bulk
- **Instalación simplificada**: pip install vs npm/node setup

## Conclusión

La migración de gitlab-mcp a Python es técnicamente viable y aportará beneficios significativos. El uso de FastMCP y python-gitlab asegura una implementación robusta que mantiene la compatibilidad completa con el proyecto original mientras aprovecha las ventajas del ecosistema Python.

**¿Estás listo para proceder con la implementación? ¿Hay algún aspecto específico que te gustaría que profundice o modifique?**