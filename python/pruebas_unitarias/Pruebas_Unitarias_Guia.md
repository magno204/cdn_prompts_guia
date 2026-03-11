# Guía de Pruebas Unitarias en Python

Este documento responde las preguntas fundamentales sobre pruebas unitarias y proporciona orientación práctica aplicable a cualquier proyecto Python.

---

## 1. ¿Cuáles son las mejores prácticas al momento de crear una prueba unitaria?

### Principios fundamentales

| Práctica | Descripción |
|----------|-------------|
| **Aislamiento** | Cada prueba debe ser independiente y no depender del orden de ejecución ni del estado de otras pruebas. |
| **Rapidez** | Las pruebas unitarias deben ejecutarse en milisegundos. Evitar I/O real (BD, APIs, colas de mensajes). |
| **Determinismo** | El mismo código y datos deben producir siempre el mismo resultado. Sin aleatoriedad ni dependencias externas variables. |
| **Una sola responsabilidad** | Cada prueba debe verificar un único comportamiento o escenario. |
| **Nombres descriptivos** | El nombre de la prueba debe describir qué se prueba y bajo qué condiciones. |
| **Arrange-Act-Assert (AAA)** | Estructurar cada prueba en tres fases: preparar datos, ejecutar la acción, verificar el resultado. |
| **Mock de dependencias** | Sustituir servicios externos (bases de datos, colas, APIs) por mocks o stubs para aislar la lógica. |
| **Cobertura significativa** | Priorizar ramas críticas, casos límite y lógica de negocio sobre cobertura numérica. |
| **Docstring obligatorio (GWT)** | Cada método de prueba **debe** tener un docstring con el patrón **Dado/Cuando/Entonces** que describa el escenario en lenguaje natural. |

### Convenciones adicionales

- **No probar código trivial**: Evitar pruebas para getters/setters o código que solo delega.
- **Pruebas como documentación**: Las pruebas deben servir como ejemplos de uso del código. El docstring con patrón Dado/Cuando/Entonces es el mecanismo principal para lograr esto.
- **Mantener pruebas simples**: Si una prueba es difícil de escribir, el código probablemente necesita refactorización.

---

## 2. ¿Qué debería "Evaluar" una prueba unitaria?

Una prueba unitaria debe evaluar que **una unidad de código** (función, método, clase) se comporta correctamente ante ciertas entradas y condiciones.

### Qué evaluar

1. **Comportamiento esperado**: Que la salida sea correcta para entradas válidas.
2. **Casos límite**: Valores vacíos, `None`, cadenas largas, números en los extremos.
3. **Casos de error**: Que se lancen las excepciones correctas con mensajes adecuados.
4. **Invariantes**: Propiedades que siempre deben cumplirse (ej. un identificador no vacío).
5. **Interacciones**: Que se llamen las dependencias con los parámetros correctos (cuando se usan mocks).

### Qué NO evaluar

- Integración entre múltiples servicios reales (eso son pruebas de integración).
- Detalles de implementación internos que pueden cambiar.
- Código de terceros (librerías externas).

---

## 3. ¿Cómo hacer que el código de una prueba unitaria sea fácil de leer y mantener?

### Estructura AAA (Arrange-Act-Assert)

```python
def test_validar_pedido_con_datos_validos():
    # Arrange: preparar datos de entrada
    pedido = {"producto": "Laptop", "cantidad": 2, "precio": 1500.00}

    # Act: ejecutar la función bajo prueba
    resultado = validar_pedido(pedido)

    # Assert: verificar el resultado
    assert resultado.is_valid is True
    assert resultado.errors == []
```

### Nombres descriptivos

- **Formato sugerido**: `test_<nombre_funcion>_<condicion>_<resultado_esperado>`
- **Ejemplo**: `test_validar_pedido_con_cantidad_negativa_devuelve_error`

### Fixtures y datos reutilizables

- Extraer datos de prueba a constantes o fixtures (pytest).
- Usar factories para crear objetos de prueba con valores por defecto sensatos.
- Colocar fixtures compartidas en `conftest.py` para que estén disponibles en todos los archivos de prueba del directorio.

### `conftest.py` para fixtures compartidas

Pytest descubre automáticamente los archivos `conftest.py` y comparte sus fixtures. Esto evita duplicar datos de prueba en cada archivo.

```
tests/
├── conftest.py              # Fixtures compartidas
├── test_validators.py
├── test_models.py
└── test_services.py
```

```python
# tests/conftest.py
"""Fixtures compartidas para todas las pruebas del proyecto."""

import pytest


@pytest.fixture
def pedido_valido():
    """Pedido que cumple todas las reglas de validación."""
    return {
        "producto": "Laptop",
        "cantidad": 2,
        "precio": 1500.00,
        "cliente_id": "CLI-001",
        "direccion": "Calle 123 #45-67",
    }


@pytest.fixture
def pedido_modelo(pedido_valido):
    """Instancia del modelo Pedido lista para usar en pruebas."""
    from src.models import Pedido
    return Pedido.from_dict(pedido_valido)
```

### Agrupar pruebas con clases

Usar clases para agrupar pruebas relacionadas mejora la navegación y permite compartir contexto:

```python
class TestValidarPedido:
    """Pruebas para la función validar_pedido."""

    def test_pedido_valido_devuelve_resultado_exitoso(self, pedido_valido):
        resultado = validar_pedido(pedido_valido)
        assert resultado.is_valid is True

    def test_pedido_vacio_devuelve_error(self):
        resultado = validar_pedido({})
        assert resultado.is_valid is False

    def test_cantidad_negativa_devuelve_error(self, pedido_valido):
        pedido_valido["cantidad"] = -1
        resultado = validar_pedido(pedido_valido)
        assert resultado.is_valid is False
```

### Pruebas parametrizadas

Cuando múltiples entradas deben producir el mismo tipo de resultado, usar `@pytest.mark.parametrize` evita duplicar funciones de prueba:

```python
@pytest.mark.parametrize("pedido_invalido,fragmento_error", [
    ({}, "vacío"),
    ({"producto": "x"}, "cantidad"),
    ({"producto": "x", "cantidad": -1, "precio": 10.0}, "negativa"),
])
def test_validar_pedido_datos_invalidos(pedido_invalido, fragmento_error):
    resultado = validar_pedido(pedido_invalido)
    assert resultado.is_valid is False
    assert any(fragmento_error in e for e in resultado.errors)
```

### Docstring obligatorio con patrón Dado/Cuando/Entonces

**Cada método de prueba debe incluir un docstring** que describa el escenario usando el patrón **Dado/Cuando/Entonces** (equivalente en español de Given/When/Then). Esto convierte las pruebas en documentación viva del comportamiento esperado del sistema.

```python
def test_validar_pedido_con_cantidad_negativa_devuelve_error(self, pedido_valido):
    """
    Dado un pedido con cantidad negativa,
    cuando se valida el pedido,
    entonces is_valid es False y el error indica que la cantidad no puede ser negativa.
    """
    # Arrange
    pedido_valido["cantidad"] = -1

    # Act
    resultado = validar_pedido(pedido_valido)

    # Assert
    assert resultado.is_valid is False
```

| Componente | Propósito | Ejemplo |
|------------|-----------|---------|
| **Dado** | Describe el estado inicial o contexto | "Dado un pedido con cantidad negativa" |
| **Cuando** | Describe la acción que se ejecuta | "cuando se valida el pedido" |
| **Entonces** | Describe el resultado esperado | "entonces is_valid es False y el error indica que la cantidad no puede ser negativa" |

> **¿Por qué es obligatorio?** El nombre del método describe _qué_ se prueba, pero el docstring explica _por qué_ y _bajo qué condiciones_. Cuando una prueba falla, el docstring permite entender el escenario sin leer el código. Además, herramientas como `pytest -v` muestran los nombres pero no los docstrings; tenerlos en el código complementa la salida del runner.

### Un concepto por prueba

- Evitar múltiples `assert` que verifiquen cosas no relacionadas.
- Si hay varios escenarios, crear una prueba por escenario.
- Excepción: varios `assert` sobre el **mismo** resultado (ej. `is_valid`, `errors`, `data`) sí son válidos porque verifican distintas facetas del mismo comportamiento.

### Comentarios solo cuando aporten

- Preferir nombres claros antes que comentarios.
- Comentar solo cuando la intención no sea obvia (ej. "Simula timeout de la base de datos").

### Organización de archivos

Estructura recomendada con carpeta `tests/` espejo:

```
mi_proyecto/
├── src/
│   ├── validators/
│   │   └── pedido_validator.py
│   ├── models/
│   │   └── pedido.py
│   └── services/
│       └── pedido_service.py
├── tests/
│   ├── conftest.py                    # Fixtures compartidas
│   ├── test_pedido_validator.py       # Pruebas de validación
│   ├── test_pedido.py                 # Pruebas de modelos
│   └── test_pedido_service.py         # Pruebas del servicio (con mocks)
├── requirements.txt
└── requirements-dev.txt               # Dependencias solo para pruebas
```

---

## 4. Ejemplo de una prueba unitaria

A continuación se muestra un ejemplo de prueba unitaria para una función de validación (`validar_pedido`), que es una función pura y fácil de probar sin mocks.

### Requisitos previos

Instalar las dependencias de desarrollo:

```bash
pip install -r requirements-dev.txt
```

Archivo **`requirements-dev.txt`** (dependencias exclusivas para pruebas, separadas de producción):

```
pytest>=7.0.0
pytest-cov>=4.0.0
pytest-mock>=3.10.0
```

### Archivo de prueba

**Ubicación sugerida**: `tests/test_pedido_validator.py`

```python
"""Pruebas unitarias para el validador de pedidos."""

import pytest

from src.validators.pedido_validator import validar_pedido


# Fixture: pedido válido reutilizable
@pytest.fixture
def pedido_valido():
    """Pedido que cumple todas las reglas de validación."""
    return {
        "producto": "Laptop",
        "cantidad": 2,
        "precio": 1500.00,
        "cliente_id": "CLI-001",
        "direccion": "Calle 123 #45-67",
    }


def test_validar_pedido_con_datos_validos_devuelve_resultado_exitoso(pedido_valido):
    """
    Dado un pedido que cumple todas las reglas de validación,
    cuando se valida el pedido,
    entonces is_valid es True y data contiene el pedido original.
    """
    # Act
    resultado = validar_pedido(pedido_valido)

    # Assert
    assert resultado.is_valid is True
    assert resultado.errors == []
    assert resultado.data == pedido_valido


def test_validar_pedido_con_datos_vacios_devuelve_error():
    """
    Dado un pedido vacío o None,
    cuando se valida el pedido,
    entonces is_valid es False y errors indica que el pedido es inválido.
    """
    # Act
    resultado = validar_pedido({})

    # Assert
    assert resultado.is_valid is False
    assert len(resultado.errors) > 0
    assert resultado.errors[0] == "El pedido está vacío o es inválido"
    assert resultado.data is None
```

### Ejecución

```bash
# Desde la raíz del proyecto
python -m pytest tests/test_pedido_validator.py -v
```

### Por qué este ejemplo es mantenible

1. **AAA claro**: Cada prueba tiene Arrange, Act y Assert bien delimitados.
2. **Fixture reutilizable**: `pedido_valido` evita duplicar datos en múltiples pruebas.
3. **Nombre descriptivo**: Indica la condición y el resultado esperado.
4. **Docstring obligatorio**: Cada prueba documenta el escenario con el patrón Dado/Cuando/Entonces. Esto es un requisito, no una sugerencia.
5. **Sin dependencias externas**: No usa bases de datos, colas ni APIs; la función es pura.
6. **Un concepto por prueba**: Cada función verifica un solo escenario.
7. **Aserción exacta**: Compara contra el mensaje de error real del código en lugar de buscar subcadenas, lo que hace la prueba más precisa y detecta cambios involuntarios.

---

## 5. Ejemplo avanzado: pruebas parametrizadas

Cuando múltiples entradas inválidas deben ser rechazadas por el mismo validador, `@pytest.mark.parametrize` permite probar todos los casos sin duplicar funciones.

**Ubicación**: `tests/test_pedido_validator.py`

```python
"""Pruebas parametrizadas para el validador de pedidos."""

import pytest

from src.validators.pedido_validator import validar_pedido


# Datos base válidos para reutilizar en variaciones
PEDIDO_BASE = {
    "producto": "Laptop",
    "cantidad": 2,
    "precio": 1500.00,
    "cliente_id": "CLI-001",
    "direccion": "Calle 123 #45-67",
}


def _modificar_pedido(cambios: dict) -> dict:
    """Crea una copia del pedido base aplicando cambios."""
    import copy
    pedido = copy.deepcopy(PEDIDO_BASE)
    pedido.update(cambios)
    return pedido


class TestValidarPedidoDatosInvalidos:
    """Pruebas para pedidos que deben ser rechazados por el validador."""

    @pytest.mark.parametrize("pedido,fragmento_error", [
        pytest.param(
            {},
            "vacío",
            id="pedido-vacio"
        ),
        pytest.param(
            {"producto": "x"},
            "cantidad",
            id="sin-cantidad"
        ),
        pytest.param(
            _modificar_pedido({"cantidad": -1}),
            "negativa",
            id="cantidad-negativa"
        ),
        pytest.param(
            _modificar_pedido({"precio": 0}),
            "precio",
            id="precio-cero"
        ),
    ])
    def test_pedido_invalido_devuelve_error(self, pedido, fragmento_error):
        """
        Dado un pedido que viola una regla de validación,
        cuando se valida el pedido,
        entonces is_valid es False y el error contiene el campo problemático.
        """
        # Act
        resultado = validar_pedido(pedido)

        # Assert
        assert resultado.is_valid is False
        assert any(fragmento_error in e for e in resultado.errors), (
            f"Se esperaba '{fragmento_error}' en los errores, pero se obtuvo: {resultado.errors}"
        )
        assert resultado.data is None
```

### Por qué este patrón es útil

- **Menos código**: Un solo cuerpo de prueba cubre 4+ escenarios.
- **IDs legibles**: `pytest.param(..., id="...")` genera nombres descriptivos en la salida de pytest.
- **Fácil de extender**: Agregar un nuevo caso inválido es agregar un `pytest.param(...)` a la lista.
- **Mensajes de error claros**: El segundo argumento de `assert` explica qué falló si la prueba no pasa.

---

## 6. Ejemplo avanzado: pruebas con mocks

Cuando una clase depende de servicios externos (bases de datos, colas de mensajes, APIs de terceros), esas dependencias **deben** ser sustituidas por mocks para mantener las pruebas unitarias rápidas y aisladas.

En este ejemplo, `OrderProcessor` depende de `DatabaseService` (base de datos) y `NotificationService` (cola de mensajes), que se reemplazan por mocks.

**Ubicación**: `tests/test_order_processor.py`

```python
"""Pruebas unitarias para el procesador de pedidos con mocks."""

import pytest
from unittest.mock import patch, MagicMock

from src.models import Order
from src.services.order_processor import OrderProcessor
from src.exceptions import DatabaseError


# --- Fixtures ---

@pytest.fixture
def orden():
    """Crea una Order válida para pruebas."""
    return Order.from_dict({
        "order_id": "ORD-001",
        "producto": "Laptop",
        "cantidad": 2,
        "precio": 1500.00,
        "cliente_id": "CLI-001",
        "estado": "PENDIENTE",
    })


# --- Pruebas ---

class TestOrderProcessor:
    """Pruebas del procesador de pedidos."""

    @patch("src.services.order_processor.NotificationService")
    @patch("src.services.order_processor.DatabaseService")
    def test_procesar_orden_nueva_guarda_y_notifica(
        self, mock_db_service, mock_notification_service, orden
    ):
        """
        Dado una orden nueva (no duplicada),
        cuando se procesa,
        entonces se guarda en la base de datos y se envía una notificación.
        """
        # Arrange
        mock_db_service.find_by_id.return_value = None
        mock_db_service.save.return_value = "db-id-001"

        processor = OrderProcessor()

        # Act
        resultado = processor.process(orden)

        # Assert
        assert resultado.success is True
        assert resultado.document_id == "db-id-001"
        assert resultado.error is None
        mock_db_service.save.assert_called_once()
        mock_notification_service.return_value.send.assert_called_once()

    @patch("src.services.order_processor.NotificationService")
    @patch("src.services.order_processor.DatabaseService")
    def test_procesar_orden_duplicada_no_guarda_en_bd(
        self, mock_db_service, mock_notification_service, orden
    ):
        """
        Dado una orden con ID ya existente en la base de datos,
        cuando se procesa,
        entonces retorna éxito pero NO guarda ni notifica.
        """
        # Arrange: simular que ya existe un registro con ese ID
        mock_db_service.find_by_id.return_value = {"order_id": "ORD-001"}

        processor = OrderProcessor()

        # Act
        resultado = processor.process(orden)

        # Assert
        assert resultado.success is True
        assert resultado.error == "Orden duplicada"
        mock_db_service.save.assert_not_called()
        mock_notification_service.return_value.send.assert_not_called()

    @patch("src.services.order_processor.NotificationService")
    @patch("src.services.order_processor.DatabaseService")
    def test_procesar_orden_con_error_de_bd_devuelve_fallo(
        self, mock_db_service, mock_notification_service, orden
    ):
        """
        Dado que la base de datos lanza un DatabaseError al guardar,
        cuando se procesa la orden,
        entonces retorna success=False con el mensaje de error.
        """
        # Arrange
        mock_db_service.find_by_id.return_value = None
        mock_db_service.save.side_effect = DatabaseError(
            message="Error de conexión", operation="save"
        )

        processor = OrderProcessor()

        # Act
        resultado = processor.process(orden)

        # Assert
        assert resultado.success is False
        assert "base de datos" in resultado.error.lower()
        mock_notification_service.return_value.send.assert_not_called()
```

### Anatomía de un mock

| Concepto | Ejemplo | Propósito |
|----------|---------|-----------|
| `@patch("ruta.completa.DatabaseService")` | Reemplaza `DatabaseService` por un `MagicMock` | Evitar conexión real a la base de datos |
| `.return_value = None` | Simular que `find_by_id` no encontró nada | Controlar el flujo de la prueba |
| `.side_effect = DatabaseError(...)` | Simular que `save` lanza una excepción | Probar el manejo de errores |
| `.side_effect = [mock_a, mock_b]` | Retornar un valor distinto en cada llamada consecutiva al mock | Diferenciar múltiples instancias del mismo constructor |
| `.assert_called_once()` | Verificar que el mock fue llamado exactamente 1 vez | Probar interacciones |
| `.assert_not_called()` | Verificar que el mock NO fue llamado | Probar que un camino no se ejecutó |

> **Regla de oro**: La ruta en `@patch` debe ser **donde se importa** el objeto, no donde se define. Si `order_processor.py` hace `from src.database.db_service import DatabaseService`, la ruta del patch es `src.services.order_processor.DatabaseService`.

### `patch.object` para mockear delegación interna

Cuando una clase delega internamente a otros métodos propios (ej. `get_valid_token` → `_create_cache` → `_fetch_new`), usar `@patch("modulo.Clase.metodo")` puede ser frágil. En su lugar, `patch.object` permite mockear un método específico de una instancia concreta:

```python
manager = TokenManager()

with patch.object(manager, "_create_cache", return_value={"access_token": "nuevo"}) as mock_create:
    result = manager.get_valid_token(request_id="abc-123", config=config)

assert result == "nuevo"
mock_create.assert_called_once()
```

#### ¿Cuándo usar cada uno?

| Técnica | Cuándo usarla | Ejemplo |
|---------|---------------|---------|
| `@patch("modulo.Dependencia")` | Mockear una **dependencia externa** importada en el módulo bajo prueba | `@patch("src.auth.token_manager.CacheService")` |
| `patch.object(instancia, "método")` | Mockear un **método interno** de la misma clase para aislar un eslabón de la cadena | `patch.object(manager, "_fetch_new", return_value=token)` |

#### Ventajas de `patch.object`

- **Aislamiento preciso**: Mockea solo el método indicado de esa instancia; el resto de la clase funciona normalmente.
- **Ideal para cadenas de delegación**: Permite probar un método intermedio sin que ejecute los métodos que delega, mientras los métodos de validación siguen ejecutándose realmente.
- **Sin dependencia de rutas de importación**: No requiere conocer la ruta completa del módulo; opera directamente sobre el objeto.

#### Ejemplo completo: probar un eslabón intermedio

```python
def test_create_cache_actualiza_config_y_persiste(self, config_base):
    """
    Dado una configuración válida,
    cuando se invoca _create_cache,
    entonces guarda el token en la configuración y persiste en el servicio de caché.
    """
    manager = TokenManager()
    request_id = "abc-123"
    token = {"access_token": "abc-token"}

    # Mock del método interno (delegación) y de la dependencia externa
    with patch.object(manager, "_fetch_new", return_value=token) as mock_fetch:
        with patch("src.auth.token_manager.CacheService.set_config") as mock_cache:
            result = manager._create_cache(request_id, config_base)

    assert result == token
    assert config_base["token"] == token
    mock_fetch.assert_called_once_with(config_base)
    mock_cache.assert_called_once()
```

> **Nota**: Combinar `patch.object` (para delegación interna) con `@patch` (para dependencias externas) en la misma prueba es un patrón válido y común.

### `side_effect` con lista para múltiples instancias del mismo constructor

Cuando una clase crea **varias instancias de la misma dependencia** en su `__init__` (ej. un servicio que crea múltiples clientes de cola), un solo `return_value` haría que todos los atributos apunten al mismo mock. Usando `side_effect` con una lista, cada llamada consecutiva al constructor retorna un mock distinto:

```python
@pytest.fixture
def service_instance():
    """Crea instancia del servicio con clientes diferenciados."""
    with patch("src.services.queue_service.QueueClient") as mock_client:
        client_a = MagicMock()
        client_b = MagicMock()
        client_c = MagicMock()
        mock_client.side_effect = [client_a, client_b, client_c]
        instance = QueueService()
    return instance
```

Esto permite verificar que cada método usa el cliente correcto:

```python
def test_send_audit_message_usa_cliente_de_auditoria(self, service_instance):
    """
    Dado un mensaje de auditoría válido,
    cuando se invoca send_audit_message,
    entonces retorna la respuesta del cliente de auditoría (no del de otros canales).
    """
    expected = {"MessageId": "audit-001"}
    service_instance._audit_client.send_message.return_value = expected

    result = service_instance.send_audit_message({"event": "ok"})

    assert result == expected
    service_instance._audit_client.send_message.assert_called_once_with({"event": "ok"})
```

#### ¿Cuándo usar este patrón?

| Escenario | Técnica |
|-----------|---------|
| La clase crea **una** instancia de la dependencia | `return_value=MagicMock()` (estándar) |
| La clase crea **múltiples** instancias de la misma dependencia | `side_effect=[mock_1, mock_2, ...]` (una por llamada) |

> **Importante**: El orden de la lista debe coincidir con el orden en que el constructor crea las instancias en `__init__`. Si el orden cambia en el código fuente, la fixture debe actualizarse.

---

## 7. Ejemplo avanzado: probar excepciones con `pytest.raises`

Cuando una función debe lanzar una excepción, `pytest.raises` permite verificar tanto el tipo como el contenido del error.

**Ubicación**: `tests/test_pedido_validator.py`

```python
"""Prueba de validar_pedido_o_lanzar que lanza excepción."""

import pytest

from src.validators.pedido_validator import validar_pedido_o_lanzar
from src.exceptions import ValidationError


class TestValidarPedidoOLanzar:
    """Pruebas para la función que lanza excepción en caso de error."""

    def test_pedido_valido_devuelve_datos(self, pedido_valido):
        """
        Dado un pedido válido,
        cuando se llama validar_pedido_o_lanzar,
        entonces devuelve el pedido sin lanzar excepción.
        """
        # Act
        resultado = validar_pedido_o_lanzar(pedido_valido)

        # Assert
        assert resultado == pedido_valido

    def test_pedido_vacio_lanza_validation_error(self):
        """
        Dado un pedido vacío,
        cuando se llama validar_pedido_o_lanzar,
        entonces lanza ValidationError con el mensaje adecuado.
        """
        # Act & Assert
        with pytest.raises(ValidationError) as exc_info:
            validar_pedido_o_lanzar({})

        assert "Validación fallida" in str(exc_info.value)

    def test_pedido_sin_producto_lanza_validation_error(self):
        """
        Dado un pedido sin el campo producto,
        cuando se llama validar_pedido_o_lanzar,
        entonces lanza ValidationError mencionando el campo faltante.
        """
        # Arrange
        pedido_sin_producto = {"cantidad": 2, "precio": 10.0}

        # Act & Assert
        with pytest.raises(ValidationError):
            validar_pedido_o_lanzar(pedido_sin_producto)
```

### Cuándo usar `pytest.raises`

- Cuando la función **debe** lanzar una excepción específica ante cierta entrada.
- Para verificar el **tipo** de excepción (`ValidationError`, no una genérica `Exception`).
- Para inspeccionar el **mensaje** o **atributos** del error usando `exc_info.value`.

---

## 8. Ejemplo avanzado: pruebas de clases Singleton

Muchas aplicaciones usan el patrón Singleton (`__new__` + `_initialized`). Probar singletons requiere cuidado especial porque el estado persiste entre pruebas si no se limpia explícitamente.

### Problema

Si una prueba instancia un singleton, la siguiente prueba reutilizará esa misma instancia (con su estado interno) porque `_instance` es un atributo de clase. Esto viola el principio de **aislamiento**.

### Solución: fixture `autouse=True` para reset del singleton

```python
@pytest.fixture(autouse=True)
def reset_singleton():
    """Restablece el singleton para aislar cada prueba."""
    MiSingleton._instance = None
    yield
    MiSingleton._instance = None
```

| Aspecto | Detalle |
|---------|---------|
| `autouse=True` | Se ejecuta **automáticamente** antes y después de cada prueba del archivo, sin necesidad de inyectarla. |
| `_instance = None` antes del `yield` | Garantiza que cada prueba arranca sin singleton previo. |
| `_instance = None` después del `yield` | Limpia el estado para que no afecte a otros archivos de prueba. |

### Fixture de instancia con dependencias mockeadas

Además del reset, es útil tener una fixture que cree la instancia con sus dependencias externas ya sustituidas por mocks:

```python
@pytest.fixture
def cache_instance():
    """Crea una instancia del singleton con dependencias mockeadas."""
    mock_client = MagicMock()
    with patch("src.cache.cache_service._create_client", return_value=mock_client):
        instance = CacheService()
    return instance
```

Esto permite que cada prueba configure el mock según su escenario:

```python
def test_get_config_con_datos_en_cache(self, cache_instance):
    """
    Dado una clave existente en el servicio de caché con datos serializados en JSON,
    cuando se consulta con get_config,
    entonces se retorna el diccionario y se guarda en caché local.
    """
    # Arrange
    payload = {"client_id": "cli-001", "active": True}
    cache_instance.client.get.return_value = json.dumps(payload)

    # Act
    result = cache_instance.get_config(key="900123")

    # Assert
    assert result == payload
    assert cache_instance._local_cache["900123"] == payload
```

### Patrón completo para cualquier singleton

```python
@pytest.fixture(autouse=True)
def reset_singleton():
    """Restablece el singleton de <NombreClase> para aislar cada prueba."""
    NombreClase._instance = None
    yield
    NombreClase._instance = None


@pytest.fixture
def nombre_instance():
    """Crea una instancia de <NombreClase> con dependencias mockeadas."""
    with patch("ruta.completa.dependencia_externa", return_value=MagicMock()):
        instance = NombreClase()
    return instance
```

### Cuándo usar este patrón

- **Siempre** que la clase bajo prueba use `__new__` con `_instance` o un mecanismo equivalente de singleton.
- **No** usar `autouse=True` para fixtures que no sean de limpieza de estado global; reservarlo para singletons y estado compartido a nivel de módulo.

---

## 9. Herramientas y configuración

### Medir cobertura con `pytest-cov`

La cobertura de código indica qué líneas y ramas del código fueron ejecutadas por las pruebas.

```bash
# Cobertura del código fuente del proyecto
python -m pytest tests/ --cov=src --cov-report=term-missing -v
```

| Flag | Propósito |
|------|-----------|
| `--cov=src` | Medir cobertura solo del código fuente (no de `tests/`) |
| `--cov-report=term-missing` | Mostrar qué líneas **no** están cubiertas |
| `--cov-report=html` | Generar reporte HTML navegable en `htmlcov/` |

#### Ejemplo de salida

```
Name                                        Stmts   Miss  Cover   Missing
---------------------------------------------------------------------------
src/validators/pedido_validator.py             30      2    93%   105, 121
src/services/order_processor.py                42      5    88%   98-104
---------------------------------------------------------------------------
TOTAL                                          72      7    90%
```

> **Meta recomendada**: 80%+ de cobertura en el código fuente. Priorizar ramas de negocio críticas sobre cobertura numérica total.

### Configuración de pytest (`pyproject.toml`)

Centralizar la configuración de pytest evita repetir flags en cada ejecución:

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short"

[tool.coverage.run]
source = ["src"]

[tool.coverage.report]
show_missing = true
fail_under = 80
```

### Integración en el flujo de CI/CD

Se recomienda ejecutar las pruebas como paso previo al despliegue en cualquier pipeline de integración continua:

```bash
# Paso 1: Ejecutar pruebas con cobertura
python -m pytest tests/ --cov=src --cov-report=term-missing

# Paso 2: Solo si las pruebas pasan, proceder con el despliegue
# (el comando de despliegue específico del proyecto)
```

En caso de contar con un pipeline CI/CD (GitHub Actions, GitLab CI, Jenkins, etc.), agregar las pruebas como paso previo al despliegue garantiza que ningún cambio con errores llegue a producción.

---

## Referencias

- [pytest — Documentación oficial](https://docs.pytest.org/)
- [unittest.mock — Documentación oficial de Python](https://docs.python.org/3/library/unittest.mock.html)
- [pytest-cov — Plugin de cobertura para pytest](https://pytest-cov.readthedocs.io/)
- [pytest.mark.parametrize — Parametrización de pruebas](https://docs.pytest.org/en/stable/how-to/parametrize.html)
- [Test-Driven Development (TDD)](https://es.wikipedia.org/wiki/Desarrollo_guiado_por_pruebas)
- [FIRST principles (Fast, Independent, Repeatable, Self-validating, Timely)](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
