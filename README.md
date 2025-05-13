# Documentación de integración RPC con Odoo para el modelo `res.partner`

---

## 1. Introducción  
Esta guía describe cómo conectar tu aplicación externa a un servidor Odoo mediante JSON-RPC, consultar (`search_read`) registros de clientes/proveedores (`res.partner`) y crear nuevos contactos. Está basada en la estructura del módulo de ejemplo que implementa `ExternalRPCConfig` y muestra las llamadas genéricas de autenticación y búsqueda.  

---

## 2. Requisitos  
- **Servidor Odoo** v13 o superior con el XML-RPC o HTTP-JSON-RPC habilitado.  
- Dependencias Python:
  ```bash
  pip install requests
  ```  
- Un registro en Odoo (`external.rpc.config`) con los datos de conexión:
  - `url` (por ejemplo, `https://mi-odoo.com`)
  - `db_name`
  - `username`
  - `password`

---

## 3. Configuración de la conexión RPC

### 3.1. Autenticación  
Se utiliza el endpoint `/web/session/authenticate` para obtener una sesión válida.

```python
import requests
import json

def get_rpc_session(url, db, username, password):
    """Autentica y devuelve un objeto requests.Session con cookies de sesión."""
    login_url = f"{url}/web/session/authenticate"
    payload = {
        "jsonrpc": "2.0",
        "method": "call",
        "params": {
            "db": db,
            "login": username,
            "password": password,
        },
        "id": 1
    }
    headers = {'Content-Type': 'application/json'}
    session = requests.Session()
    response = session.post(login_url, data=json.dumps(payload), headers=headers)
    result = response.json()
    if result.get('error'):
        raise Exception(f"Error de autenticación: {result['error']['message']}")
    return session
```

- **Parámetros**  
  - `url`: URL base del servidor Odoo.  
  - `db`: nombre de la base de datos.  
  - `username`, `password`: credenciales de un usuario válido.  
- **Retorno**:  
  - `requests.Session()` autenticada, lista para llamadas posteriores.

---

## 4. Consultar contactos (`search_read`)

### 4.1. Método genérico de búsqueda  
Utilizamos el endpoint `/web/dataset/call_kw` con `method=search_read`.

```python
def search_data(session, url, model, fields, domain=None):
    """
    Llama al RPC search_read y devuelve una lista de diccionarios con los campos solicitados.
    """
    endpoint = f"{url}/web/dataset/call_kw"
    payload = {
        "jsonrpc": "2.0",
        "method": "call",
        "params": {
            "model": model,
            "method": "search_read",
            "args": [domain or [], fields],
            "kwargs": {}
        },
        "id": 2
    }
    headers = {'Content-Type': 'application/json'}
    response = session.post(endpoint, data=json.dumps(payload), headers=headers)
    result = response.json()
    if result.get('error'):
        raise Exception(f"Error al consultar {model}: {result['error']['message']}")
    return result['result']
```

### 4.2. Ejemplo práctico: obtener todos los partners verificados  
```python
# Campos que deseamos obtener de res.partner
FIELDS = ['id', 'name', 'email', 'vat', 'is_company']

# Dominio opcional: solo partners activos (state='active')
DOMAIN = [('active', '=', True)]

# Uso
session = get_rpc_session(URL, DB, USER, PASS)
partners = search_data(session, URL, 'res.partner', FIELDS, DOMAIN)

for p in partners:
    print(f"{p['id']}: {p['name']} – VAT: {p.get('vat')}")
```

---

## 5. Crear un nuevo contacto

### 5.1. Llamada `create` vía RPC  
Para crear, cambiamos `method` a `create` y pasamos un solo diccionario con los valores:

```python
def create_data(session, url, model, vals):
    """
    Llama al RPC create y devuelve el ID del nuevo registro.
    """
    endpoint = f"{url}/web/dataset/call_kw"
    payload = {
        "jsonrpc": "2.0",
        "method": "call",
        "params": {
            "model": model,
            "method": "create",
            "args": [vals],
            "kwargs": {}
        },
        "id": 3
    }
    headers = {'Content-Type': 'application/json'}
    response = session.post(endpoint, data=json.dumps(payload), headers=headers)
    result = response.json()
    if result.get('error'):
        raise Exception(f"Error al crear {model}: {result['error']['message']}")
    return result['result']  # ID del nuevo partner
```

### 5.2. Ejemplo práctico: crear un partner  
```python
new_partner_vals = {
    'name': "ACME S.A.",
    'email': "contacto@acme.com",
    'vat': "J123456789",
    'is_company': True,
}

partner_id = create_data(session, URL, 'res.partner', new_partner_vals)
print(f"Partner creado con ID {partner_id}")
```

---

## 6. Manejo de errores y validaciones  
- Verifica siempre la existencia de `result['error']` en la respuesta JSON.  
- Captura excepciones de red o JSON corrupto con `try/except`.


---

## 7. Resumen de pasos  
1. **Autenticarse** → obtener sesión.  
2. **Consultar** → `search_read`.  
3. **Crear** → `create`.
