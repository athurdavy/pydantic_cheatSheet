# Pydantic v2 – Cheat Sheet (ES)
*(Actualizado: 2025-09-02)*

---

## Instalación
*Cómo instalar Pydantic y, opcionalmente, pydantic-settings para leer configuración desde `.env`.*
```bash
pip install pydantic
# (Opcional para configuración via .env)
pip install pydantic-settings
```

---

## Modelo básico
*Define modelos tipados con validación automática y valores por defecto.*
```python
from pydantic import BaseModel, Field

class User(BaseModel):
    id: int
    name: str = Field(..., min_length=1)
    age: int | None = Field(default=None, ge=0)
    tags: list[str] = Field(default_factory=list)

u = User(id=1, name="Ana")
```

---

## Campos y `Field`
*Restringe y documenta campos (rangos, longitudes, alias) y usa factories para generar valores por defecto.*
```python
from datetime import datetime
from uuid import UUID, uuid4
from pydantic import BaseModel, Field

class Item(BaseModel):
    uid: UUID = Field(default_factory=uuid4)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    title: str = Field(..., min_length=3, max_length=50)
    price: float = Field(..., gt=0)
    sku: str = Field(alias="SKU")  # alias de entrada/salida

i = Item(SKU="ABC-1", title="Teclado", price=49.9)
i.model_dump(by_alias=True)  # {'SKU': 'ABC-1', ...}
```

---

## Validadores (v2)
*Lógica personalizada para normalizar o validar datos más allá de las reglas básicas.*
### De campo
*Se ejecutan por campo; ideales para normalizar o castear antes/después de la validación.*
```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    email: str

    @field_validator("email", mode="before")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        return v.strip().lower()
```
### De modelo
*Validan relaciones entre varios campos (coherencia interna del modelo).*
```python
from pydantic import BaseModel, model_validator

class Range(BaseModel):
    start: int
    end: int

    @model_validator(mode="after")
    def check_order(self):
        if self.end <= self.start:
            raise ValueError("end debe ser > start")
        return self
```

---

## Campos calculados
*Propiedades derivadas que se calculan a partir de otros campos y aparecen en el dump.*
```python
from pydantic import BaseModel, computed_field

class Rectangle(BaseModel):
    w: float
    h: float

    @computed_field
    @property
    def area(self) -> float:
        return self.w * self.h
```
---

## Serialización / parseo
*Convierte modelos a dict/JSON y valida objetos o JSON entrantes para crear instancias seguras.*
```python
m = User(email="a@b.com")
m.model_dump()             # dict
m.model_dump_json()        # JSON string
User.model_validate({"email": "a@b.com"})     # desde dict/obj
User.model_validate_json('{"email": "a@b.com"}')  # desde JSON
```

---

## Adaptadores de tipo (validar sin modelo)
*Valida datos contra un tipo sin definir un modelo completo (útil para listas, uniones, etc.).*
```python
from pydantic import TypeAdapter

TA_IntList = TypeAdapter(list[int])
TA_IntList.validate_python(["1", 2, 3])  # [1, 2, 3]
TA_IntList.validate_json("[1,2,3]")      # [1, 2, 3]
```

---

## Tipos útiles
*Tipos listos para email, URLs, direcciones IP y secretos; validan formato y mejoran la seguridad.*
```python
from pydantic import BaseModel, EmailStr, HttpUrl, AnyUrl, IPvAnyAddress, SecretStr

class Cfg(BaseModel):
    email: EmailStr
    homepage: HttpUrl
    db_url: AnyUrl
    ip: IPvAnyAddress
    api_key: SecretStr
```
*(Nota: `SecretStr` oculta el secreto en logs/representaciones.)*

---

## Uniones (discriminadas)
*Soporta variantes de modelos; un campo “discriminador” decide qué forma concreta tiene el dato.*
```python
from pydantic import BaseModel
from typing import Literal, Union

class Cat(BaseModel):
    type: Literal["cat"] = "cat"
    lives: int

class Dog(BaseModel):
    type: Literal["dog"] = "dog"
    breed: str

Pet = Union[Cat, Dog]  # 'type' discrimina

def describe(p: Pet): ...
```

---

## Config del modelo (`model_config`)
*Opciones globales del modelo: manejo de campos extra, inmutabilidad, validación al asignar, etc.*
```python
from pydantic import BaseModel, ConfigDict

class M(BaseModel):
    model_config = ConfigDict(
        extra="ignore",            # "forbid" | "allow" | "ignore"
        frozen=False,              # inmutable si True
        validate_assignment=True,  # revalida al asignar
        str_strip_whitespace=True  # recorta strings
    )
    name: str
```
*(Usa `extra="forbid"` para rechazar campos desconocidos.)*

---

## Constraints con `Annotated` + `Field`
*Aplica restricciones a tipos básicos (rangos, longitudes) mediante anotaciones.*
```python
from pydantic import BaseModel, Field
from typing_extensions import Annotated  # (py<3.12)

Port = Annotated[int, Field(ge=1, le=65535)]

class Srv(BaseModel):
    host: str = Field(min_length=1)
    port: Port
```

---

## Manejo de errores
*Captura `ValidationError` para inspeccionar y reportar fallos de validación de forma estructurada.*
```python
from pydantic import BaseModel, ValidationError

class U(BaseModel):
    age: int

try:
    U(age="x")
except ValidationError as e:
    print(e.errors())  # lista detallada
```

---

## Dump y copia selectiva
*Controla qué campos serializar y cómo actualizar instancias de forma segura.*
```python
m = Item(SKU="S1", title="X", price=10)
m.model_dump(include={"title", "price"})
m.model_dump(exclude_none=True)
m.model_copy(update={"price": 12.5})
```

---

## `pydantic-settings` (cheat rápido)
*Carga configuración desde variables de entorno/`.env`, con tipado y validación de campos.*
```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "MyApp"
    debug: bool = False
    api_key: str

    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="MYAPP_",   # acepta MYAPP_API_KEY, etc.
        extra="ignore"
    )

settings = Settings()
```

---

## Listas desde CSV (string → list[str])
*Convierte cadenas separadas por comas del `.env` a listas tipadas automáticamente.*
```python
from pydantic import BaseModel, field_validator

class C(BaseModel):
    origins: list[str] = []

    @field_validator("origins", mode="before")
    @classmethod
    def split_csv(cls, v):
        if isinstance(v, str):
            return [x.strip() for x in v.split(",") if x.strip()]
        return v
```

---

## JSON Schema
*Genera el esquema JSON del modelo (útil para documentación y validación externa).*
```python
User.model_json_schema()  # dict JSON Schema del modelo
```

---

## Diferencias clave v1 → v2
*Resumen de cambios de API para migrar mentalmente de Pydantic v1 a v2.*
- `@validator` → **`@field_validator`**
- `@root_validator` → **`@model_validator`**
- `dict()/json()` → **`model_dump()` / `model_dump_json()`**
- `parse_obj/parse_raw` → **`model_validate()` / `model_validate_json()`**
- Más uso de **`typing.Annotated` + `Field`** para constraints.

---

### Tips rápidos
*Consejos prácticos para proyectos reales con Pydantic y FastAPI.*
- Usa `SecretStr` para evitar imprimir claves en claro.
- `extra="forbid"` en `model_config` endurece la validación.
- `validate_assignment=True` revalida al cambiar atributos.
- Prefiere `EmailStr`, `HttpUrl`, `AnyUrl`, `IPvAnyAddress` cuando aplique.
- Para FastAPI, combina con `BaseSettings` e inyecta con `Depends`.
