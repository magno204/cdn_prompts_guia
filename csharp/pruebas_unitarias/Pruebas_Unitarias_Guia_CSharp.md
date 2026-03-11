# Guía de Pruebas Unitarias en C# y .NET

Este documento responde las preguntas fundamentales sobre pruebas unitarias y proporciona orientación práctica aplicable a cualquier proyecto C# y .NET.

> **Nota sobre idioma en los ejemplos**: Esta guía usa nombres en español para clases de dominio (`PedidoValidator`, `Pedido`) y en inglés para patrones técnicos conocidos (`OrderProcessor`, `Setup`, `Verify`). En tu proyecto, elige una convención (español o inglés) y mantén la consistencia.

---

## 1. ¿Cuáles son las mejores prácticas al momento de crear una prueba unitaria?

### Principios fundamentales

| Práctica | Descripción |
|----------|-------------|
| **Aislamiento** | Cada prueba debe ser independiente y no depender del orden de ejecución ni del estado de otras pruebas. |
| **Rapidez** | Las pruebas unitarias deben ejecutarse en milisegundos. Evitar I/O real (BD, APIs, sistema de archivos). |
| **Determinismo** | El mismo código y datos deben producir siempre el mismo resultado. Sin aleatoriedad ni dependencias externas variables. |
| **Una sola responsabilidad** | Cada prueba debe verificar un único comportamiento o escenario. |
| **Nombres descriptivos** | El nombre de la prueba debe describir qué se prueba y bajo qué condiciones. Formato: `Metodo_Escenario_ResultadoEsperado`. |
| **Arrange-Act-Assert (AAA)** | Estructurar cada prueba en tres fases: preparar datos, ejecutar la acción, verificar el resultado. |
| **Mock de dependencias** | Sustituir servicios externos (bases de datos, APIs, sistema de archivos) por mocks o stubs para aislar la lógica. |
| **Cobertura significativa** | Priorizar ramas críticas, casos límite y lógica de negocio sobre cobertura numérica. |
| **Comentario GWT opcional** | Considerar un comentario con el patrón **Dado/Cuando/Entonces** para documentar el escenario en lenguaje natural. |

### Características de buenas pruebas (FIRST)

- **F**ast: Ejecución en milisegundos.
- **I**ndependent: Sin dependencias entre pruebas.
- **R**epeatable: Mismo resultado siempre.
- **S**elf-validating: Pasan o fallan automáticamente.
- **T**imely: No requieren tiempo desproporcionado para escribirlas.

### Convenciones adicionales

- **No probar código trivial**: Evitar pruebas para propiedades autoimplementadas o código que solo delega.
- **Pruebas como documentación**: Las pruebas deben servir como ejemplos de uso del código.
- **Mantener pruebas simples**: Si una prueba es difícil de escribir, el código probablemente necesita refactorización.
- **Evitar dependencias de infraestructura**: No usar BD real, APIs externas ni sistema de archivos en pruebas unitarias.

---

## 2. ¿Cuáles son las mejores prácticas al momento de crear una prueba unitaria?

Una prueba unitaria debe evaluar que **una unidad de código** (método, clase) se comporta correctamente ante ciertas entradas y condiciones.

### Qué evaluar

1. **Comportamiento esperado**: Que la salida sea correcta para entradas válidas.
2. **Casos límite**: Valores vacíos, `null`, cadenas largas, números en los extremos.
3. **Casos de error**: Que se lancen las excepciones correctas con mensajes adecuados.
4. **Invariantes**: Propiedades que siempre deben cumplirse (ej. un identificador no vacío).
5. **Interacciones**: Que se llamen las dependencias con los parámetros correctos (cuando se usan mocks).

### Qué NO evaluar

- Integración entre múltiples servicios reales (eso son pruebas de integración).
- Detalles de implementación internos que pueden cambiar.
- Código de terceros (librerías externas).
- Métodos privados directamente (probar a través del método público que los invoca).

---

## 3. ¿Cómo hacer que el código de una prueba unitaria sea fácil de leer y mantener?

### Estructura AAA (Arrange-Act-Assert)

```csharp
[Fact]
public void ValidarPedido_ConDatosValidos_DevuelveResultadoExitoso()
{
    // Arrange: preparar datos de entrada
    var pedido = new Pedido
    {
        Producto = "Laptop",
        Cantidad = 2,
        Precio = 1500.00m
    };

    // Act: ejecutar el método bajo prueba
    var resultado = _validador.Validar(pedido);

    // Assert: verificar el resultado
    resultado.EsValido.Should().BeTrue();
    resultado.Errores.Should().BeEmpty();
}
```

### Convención de nombres

- **Formato recomendado**: `Metodo_Escenario_ResultadoEsperado`
- **Ejemplos**:
  - `Sumar_UnSoloNumero_DevuelveElMismoNumero`
  - `ValidarPedido_ConCantidadNegativa_DevuelveError`
  - `ProcesarOrden_OrdenDuplicada_NoGuardaEnBaseDeDatos`

### Métodos auxiliares en lugar de Setup/Teardown

xUnit no incluye `[SetUp]` ni `[TearDown]`. En su lugar, usa el constructor para dependencias compartidas y métodos auxiliares para crear objetos de prueba:

```csharp
public class PedidoValidatorTests
{
    private readonly PedidoValidator _validador;

    public PedidoValidatorTests()
    {
        _validador = new PedidoValidator();
    }

    private static Pedido CrearPedidoValido()
    {
        return new Pedido
        {
            Producto = "Laptop",
            Cantidad = 2,
            Precio = 1500.00m,
            ClienteId = "CLI-001",
            Direccion = "Calle 123 #45-67"
        };
    }

    [Fact]
    public void ValidarPedido_ConDatosValidos_DevuelveResultadoExitoso()
    {
        var pedido = CrearPedidoValido();

        var resultado = _validador.Validar(pedido);

        resultado.EsValido.Should().BeTrue();
    }
}
```

### Pruebas parametrizadas con [Theory]

Cuando múltiples entradas deben producir el mismo tipo de resultado, usar `[Theory]` evita duplicar métodos de prueba.

#### [InlineData] — para tipos primitivos

```csharp
[Theory]
[InlineData("", 0)]
[InlineData(",", 0)]
[InlineData("0", 0)]
[InlineData("1,2", 3)]
[InlineData("1,2,3", 6)]
public void Sumar_ConEntradaValida_DevuelveSumaCorrecta(string input, int expected)
{
    var calculator = new StringCalculator();

    var actual = calculator.Sumar(input);

    Assert.Equal(expected, actual);
}
```

#### [MemberData] — para datos complejos (objetos, listas)

Cuando los datos de prueba son objetos que no caben en `[InlineData]`, usar `[MemberData]` con una propiedad estática:

```csharp
public static IEnumerable<object[]> PedidosInvalidos =>
    new List<object[]>
    {
        new object[] { new Pedido { Cantidad = -1 }, "cantidad" },
        new object[] { new Pedido { Producto = "" }, "producto" },
        new object[] { new Pedido { Precio = 0 }, "precio" },
    };

[Theory]
[MemberData(nameof(PedidosInvalidos))]
public void Validar_ConPedidoInvalido_DevuelveErrorEsperado(Pedido pedido, string campoError)
{
    var resultado = _validador.Validar(pedido);

    resultado.EsValido.Should().BeFalse();
    resultado.Errores.Should().Contain(e => e.Contains(campoError));
}
```

#### [ClassData] — para datos reutilizables entre varias clases de prueba

```csharp
public class PedidosInvalidosData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { new Pedido { Cantidad = -1 }, "cantidad" };
        yield return new object[] { new Pedido { Producto = "" }, "producto" };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(PedidosInvalidosData))]
public void Validar_ConPedidoInvalido_DevuelveErrorEsperado(Pedido pedido, string campoError)
{
    // ...
}
```

### Comentario con patrón Dado/Cuando/Entonces

Opcional pero recomendado para documentar escenarios complejos:

```csharp
[Fact]
public void ValidarPedido_ConCantidadNegativa_DevuelveError()
{
    // Dado un pedido con cantidad negativa,
    // cuando se valida el pedido,
    // entonces EsValido es False y el error indica que la cantidad no puede ser negativa.
    var pedido = CrearPedidoValido();
    pedido.Cantidad = -1;

    var resultado = _validador.Validar(pedido);

    resultado.EsValido.Should().BeFalse();
    resultado.Errores.Should().Contain(e => e.Contains("cantidad"));
}
```

### Evitar cadenas mágicas

Usar constantes para valores que requieren contexto:

```csharp
[Fact]
public void Sumar_NumeroMayorAlMaximo_LanzaOverflowException()
{
    const string NumeroMaximo = "1001";
    var calculator = new StringCalculator();

    var act = () => calculator.Sumar(NumeroMaximo);

    Assert.Throws<OverflowException>(act);
}
```

### Evitar lógica en las pruebas

No usar `if`, `for`, `while` ni `switch` en las pruebas. Si hace falta lógica, dividir en varias pruebas o usar `[Theory]`:

```csharp
// ❌ Evitar
[Fact]
public void Sumar_MultiplesNumeros_DevuelveResultadosCorrectos()
{
    var calculator = new StringCalculator();
    var expected = 0;
    foreach (var test in new[] { "0,0,0", "0,1,2", "1,2,3" })
    {
        Assert.Equal(expected, calculator.Sumar(test));
        expected += 3;
    }
}

// ✅ Preferir
[Theory]
[InlineData("0,0,0", 0)]
[InlineData("0,1,2", 3)]
[InlineData("1,2,3", 6)]
public void Sumar_ConEntradaValida_DevuelveSumaCorrecta(string input, int expected)
{
    var calculator = new StringCalculator();
    var actual = calculator.Sumar(input);
    Assert.Equal(expected, actual);
}
```

### Agrupar asserts con AssertionScope

Cuando una prueba tiene múltiples asserts, si el primero falla los demás no se ejecutan. `AssertionScope` de FluentAssertions permite ver **todos** los fallos a la vez:

```csharp
[Fact]
public void Validar_ConPedidoValido_DevuelveResultadoCompleto()
{
    var pedido = CrearPedidoValido();

    var resultado = _validador.Validar(pedido);

    using (new AssertionScope())
    {
        resultado.EsValido.Should().BeTrue();
        resultado.Errores.Should().BeEmpty();
        resultado.Datos.Should().NotBeNull();
        resultado.Datos.Should().BeEquivalentTo(pedido);
    }
}
```

### Organización de archivos

Estructura recomendada con carpeta de pruebas espejo:

```
MiProyecto/
├── src/
│   ├── MiProyecto/
│   │   ├── Validators/
│   │   │   └── PedidoValidator.cs
│   │   ├── Models/
│   │   │   └── Pedido.cs
│   │   └── Services/
│   │       └── PedidoService.cs
│   └── MiProyecto.csproj
├── tests/
│   ├── MiProyecto.Tests/
│   │   ├── Validators/
│   │   │   └── PedidoValidatorTests.cs
│   │   ├── Services/
│   │   │   └── PedidoServiceTests.cs
│   │   └── MiProyecto.Tests.csproj
│   └── MiProyecto.IntegrationTests/   # Pruebas de integración separadas
└── MiProyecto.sln
```

---

## 4. Frameworks de pruebas en .NET

| Framework | Atributos | Características |
|-----------|-----------|-----------------|
| **xUnit** | `[Fact]`, `[Theory]`, `[InlineData]` | Estándar de facto en .NET Core/5+. Constructor e `IDisposable` para setup/teardown. |
| **NUnit** | `[Test]`, `[TestCase]`, `[SetUp]`, `[TearDown]` | Popular en proyectos grandes. |
| **MSTest** | `[TestMethod]`, `[DataRow]` | Integración estrecha con Visual Studio. |

**Recomendación**: xUnit para proyectos nuevos en .NET 5+.

### Librerías de mocking

| Librería | Sintaxis | Notas |
|----------|----------|-------|
| **Moq** | Lambdas: `mock.Setup(x => x.Metodo()).Returns(...)` | La más popular históricamente. En v4.20.0 incluyó SponsorLink (recolección de datos), lo que generó desconfianza. Versiones posteriores lo removieron, pero muchos equipos migraron. |
| **NSubstitute** | Directa: `sub.Metodo().Returns(...)` | Sintaxis más limpia sin lambdas. Buena alternativa a Moq. |
| **FakeItEasy** | Fluida: `A.CallTo(() => fake.Metodo()).Returns(...)` | API descriptiva, fácil de aprender. |

### Librerías de aserciones

| Librería | Notas |
|----------|-------|
| **FluentAssertions 6.x** | Última versión gratuita (6.12.2). Sintaxis expresiva: `resultado.Should().BeTrue()`. |
| **FluentAssertions 7.x** | ⚠️ Requiere licencia de pago para uso comercial. Evalúa si tu proyecto lo justifica. |
| **Shouldly** | Alternativa gratuita: `resultado.ShouldBeTrue()`. Mensajes de error descriptivos. |
| **Assert (xUnit)** | Incluido con xUnit, sin dependencias adicionales: `Assert.True(resultado)`. |

---

## 5. Ejemplo de una prueba unitaria básica

### Requisitos previos

Crear un proyecto de pruebas:

```bash
dotnet new xunit -n MiProyecto.Tests -o tests/MiProyecto.Tests
dotnet add tests/MiProyecto.Tests/MiProyecto.Tests.csproj reference src/MiProyecto/MiProyecto.csproj
dotnet add tests/MiProyecto.Tests/MiProyecto.Tests.csproj package FluentAssertions --version 6.12.2
```

### Archivo de prueba

**Ubicación**: `tests/MiProyecto.Tests/Validators/PedidoValidatorTests.cs`

```csharp
using FluentAssertions;
using Xunit;

namespace MiProyecto.Tests.Validators;

public class PedidoValidatorTests
{
    private readonly PedidoValidator _validador = new();

    [Fact]
    public void Validar_ConPedidoValido_DevuelveResultadoExitoso()
    {
        // Arrange
        var pedido = new Pedido
        {
            Producto = "Laptop",
            Cantidad = 2,
            Precio = 1500.00m,
            ClienteId = "CLI-001",
            Direccion = "Calle 123 #45-67"
        };

        // Act
        var resultado = _validador.Validar(pedido);

        // Assert
        resultado.EsValido.Should().BeTrue();
        resultado.Errores.Should().BeEmpty();
        resultado.Datos.Should().BeEquivalentTo(pedido);
    }

    [Fact]
    public void Validar_ConPedidoVacio_DevuelveError()
    {
        var pedido = new Pedido();

        var resultado = _validador.Validar(pedido);

        resultado.EsValido.Should().BeFalse();
        resultado.Errores.Should().NotBeEmpty();
        resultado.Errores.Should().Contain("El pedido está vacío o es inválido");
        resultado.Datos.Should().BeNull();
    }
}
```

### Ejecución

```bash
dotnet test tests/MiProyecto.Tests/MiProyecto.Tests.csproj --verbosity normal
```

---

## 6. Ejemplo avanzado: pruebas con mocks

Cuando una clase depende de servicios externos, sustituir esas dependencias por mocks. Este ejemplo usa **Moq**, pero la lógica es equivalente con NSubstitute o FakeItEasy:

```csharp
using FluentAssertions;
using Moq;
using Xunit;

namespace MiProyecto.Tests.Services;

public class ProcesadorOrdenTests
{
    private readonly Mock<IServicioBD> _mockDb;
    private readonly Mock<IServicioNotificacion> _mockNotificacion;
    private readonly ProcesadorOrden _procesador;

    public ProcesadorOrdenTests()
    {
        _mockDb = new Mock<IServicioBD>();
        _mockNotificacion = new Mock<IServicioNotificacion>();
        _procesador = new ProcesadorOrden(_mockDb.Object, _mockNotificacion.Object);
    }

    [Fact]
    public void Procesar_OrdenNueva_GuardaYNotifica()
    {
        // Dado una orden nueva (no duplicada),
        // cuando se procesa,
        // entonces se guarda en la base de datos y se envía una notificación.
        var orden = new Orden { OrdenId = "ORD-001", Producto = "Laptop", Cantidad = 2 };
        _mockDb.Setup(db => db.BuscarPorId("ORD-001")).Returns((Orden?)null);
        _mockDb.Setup(db => db.Guardar(It.IsAny<Orden>())).Returns("db-id-001");

        var resultado = _procesador.Procesar(orden);

        resultado.Exitoso.Should().BeTrue();
        resultado.DocumentoId.Should().Be("db-id-001");
        _mockDb.Verify(db => db.Guardar(It.IsAny<Orden>()), Times.Once);
        _mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Once);
    }

    [Fact]
    public void Procesar_OrdenDuplicada_NoGuardaNiNotifica()
    {
        var orden = new Orden { OrdenId = "ORD-001" };
        _mockDb.Setup(db => db.BuscarPorId("ORD-001")).Returns(orden);

        var resultado = _procesador.Procesar(orden);

        resultado.Exitoso.Should().BeTrue();
        resultado.Error.Should().Contain("duplicada");
        _mockDb.Verify(db => db.Guardar(It.IsAny<Orden>()), Times.Never);
        _mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Never);
    }

    [Fact]
    public void Procesar_ErrorDeBaseDeDatos_DevuelveFallo()
    {
        var orden = new Orden { OrdenId = "ORD-001" };
        _mockDb.Setup(db => db.BuscarPorId("ORD-001")).Returns((Orden?)null);
        _mockDb.Setup(db => db.Guardar(It.IsAny<Orden>()))
            .Throws(new ExcepcionBaseDeDatos("Error de conexión"));

        var resultado = _procesador.Procesar(orden);

        resultado.Exitoso.Should().BeFalse();
        resultado.Error.Should().NotBeNullOrEmpty();
        _mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Never);
    }
}
```

### Anatomía de Moq

| Concepto | Ejemplo | Propósito |
|----------|---------|-----------|
| `Mock<T>()` | `new Mock<IServicioBD>()` | Crear un mock de una interfaz |
| `.Setup(...).Returns(...)` | Simular retorno de un método | Controlar el flujo de la prueba |
| `.Setup(...).Throws(...)` | Simular que un método lanza excepción | Probar manejo de errores |
| `It.IsAny<T>()` | Ignorar el valor del parámetro | Verificar solo que se llamó |
| `It.Is<T>(predicate)` | Verificar parámetro específico | Validar argumentos |
| `.Verify(..., Times.Once)` | Verificar que se llamó exactamente 1 vez | Probar interacciones |
| `.Object` | Obtener la instancia mockeada | Inyectar en el sistema bajo prueba |

### ⚠️ Cuidado con la sobre-verificación

Abusar de `Verify` acopla las pruebas a los detalles de implementación, haciéndolas frágiles ante refactorizaciones. Usa `Verify` **solo** cuando la interacción es el comportamiento que estás probando (ej. "debe enviar una notificación"), no como verificación rutinaria de cada llamada interna:

```csharp
// ❌ Sobre-verificación: prueba frágil acoplada a implementación
_mockDb.Verify(db => db.Guardar(It.IsAny<Orden>()), Times.Once);
_mockDb.Verify(db => db.BuscarPorId(It.IsAny<string>()), Times.Once);
_mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Once);
_mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Once);

// ✅ Verificar solo lo relevante al escenario
resultado.Exitoso.Should().BeTrue();
_mockNotificacion.Verify(n => n.Enviar(It.IsAny<string>()), Times.Once);
```

---

## 7. Probar excepciones con Assert.Throws

```csharp
[Fact]
public void ValidarOLanzar_ConPedidoVacio_LanzaValidationException()
{
    var validador = new PedidoValidator();
    var pedido = new Pedido();

    var act = () => validador.ValidarOLanzar(pedido);

    var exception = Assert.Throws<ValidationException>(act);
    exception.Message.Should().Contain("Validación fallida");
}
```

---

## 8. Pruebas de código asíncrono

En .NET moderno la mayoría de operaciones de I/O son asíncronas. Las pruebas deben reflejar esto:

### Prueba async básica

```csharp
[Fact]
public async Task ProcesarAsync_OrdenValida_DevuelveExito()
{
    var orden = CrearOrdenValida();
    _mockDb.Setup(db => db.GuardarAsync(It.IsAny<Orden>()))
        .ReturnsAsync("db-id-001");

    var resultado = await _procesador.ProcesarAsync(orden);

    resultado.Exitoso.Should().BeTrue();
    resultado.DocumentoId.Should().Be("db-id-001");
}
```

### Excepciones en código async

Usar `Assert.ThrowsAsync` en lugar de `Assert.Throws`:

```csharp
[Fact]
public async Task ProcesarAsync_ErrorDeConexion_LanzaExcepcion()
{
    _mockDb.Setup(db => db.GuardarAsync(It.IsAny<Orden>()))
        .ThrowsAsync(new ExcepcionBaseDeDatos("Timeout"));

    var act = () => _procesador.ProcesarAsync(new Orden());

    await Assert.ThrowsAsync<ExcepcionBaseDeDatos>(act);
}
```

### Setup async con Moq

| Método sync | Equivalente async |
|-------------|-------------------|
| `.Returns(valor)` | `.ReturnsAsync(valor)` |
| `.Throws(excepcion)` | `.ThrowsAsync(excepcion)` |
| `Assert.Throws<T>(act)` | `await Assert.ThrowsAsync<T>(act)` |

> **Importante**: El método de prueba debe retornar `Task` (no `void`) y usar `async`/`await`. xUnit maneja `Task` automáticamente.

---

## 9. Manejar referencias estáticas (DateTime.Now, etc.)

### Opción recomendada: TimeProvider (.NET 8+)

Desde .NET 8, la clase abstracta `System.TimeProvider` reemplaza la necesidad de crear interfaces personalizadas:

```csharp
// Código de producción — recibe TimeProvider por inyección
public class CalculadoraPrecio(TimeProvider timeProvider)
{
    public int ObtenerPrecioConDescuento(int precio)
    {
        if (timeProvider.GetLocalNow().DayOfWeek == DayOfWeek.Tuesday)
            return precio / 2;
        return precio;
    }
}

// Prueba — usa FakeTimeProvider del paquete Microsoft.Extensions.TimeProvider.Testing
[Fact]
public void ObtenerPrecioConDescuento_Martes_DevuelveMitadDelPrecio()
{
    var fakeTime = new FakeTimeProvider(
        new DateTimeOffset(2024, 3, 12, 0, 0, 0, TimeSpan.Zero)); // Martes
    var calculadora = new CalculadoraPrecio(fakeTime);

    var resultado = calculadora.ObtenerPrecioConDescuento(100);

    Assert.Equal(50, resultado);
}
```

```bash
dotnet add package Microsoft.Extensions.TimeProvider.Testing
```

### Opción legacy: interfaz personalizada (< .NET 8)

Para proyectos en .NET 6/7 o anteriores, crear una costura (seam) mediante una interfaz:

```csharp
public interface IProveedorFechaHora
{
    DateTime Ahora { get; }
}

public class CalculadoraPrecio
{
    public int ObtenerPrecioConDescuento(int precio, IProveedorFechaHora fechaHora)
    {
        if (fechaHora.Ahora.DayOfWeek == DayOfWeek.Tuesday)
            return precio / 2;
        return precio;
    }
}

[Fact]
public void ObtenerPrecioConDescuento_Martes_DevuelveMitadDelPrecio()
{
    var mockFecha = new Mock<IProveedorFechaHora>();
    mockFecha.Setup(d => d.Ahora).Returns(new DateTime(2024, 3, 12)); // Martes

    var calculadora = new CalculadoraPrecio();
    var resultado = calculadora.ObtenerPrecioConDescuento(100, mockFecha.Object);

    Assert.Equal(50, resultado);
}
```

---

## 10. Terminología: Fake, Stub y Mock

| Término | Uso |
|---------|-----|
| **Fake** | Término genérico para objeto que sustituye una dependencia. Puede ser stub o mock según el contexto. |
| **Stub** | Proporciona datos/configuración para que el código bajo prueba funcione. No se usa en Assert. |
| **Mock** | Además de proporcionar datos, se usa en Assert para verificar interacciones (llamadas, parámetros). |

**Regla**: Si haces `Assert` sobre el objeto fake (ej. `mock.Verify(...)`), es un **mock**. Si solo lo pasas para satisfacer dependencias, es un **stub**.

---

## 11. Diagnóstico con ITestOutputHelper

xUnit no escribe a `Console.WriteLine`. Para emitir mensajes de diagnóstico durante la ejecución de pruebas, inyectar `ITestOutputHelper`:

```csharp
public class PedidoValidatorTests(ITestOutputHelper output)
{
    [Fact]
    public void Validar_ConPedidoComplejo_DevuelveResultadoEsperado()
    {
        var pedido = CrearPedidoValido();
        output.WriteLine("Probando pedido: {0}, cantidad: {1}", pedido.Producto, pedido.Cantidad);

        var resultado = _validador.Validar(pedido);

        output.WriteLine("Resultado: EsValido={0}, Errores={1}",
            resultado.EsValido, resultado.Errores.Count);
        resultado.EsValido.Should().BeTrue();
    }
}
```

La salida se muestra en el log de cada prueba individual al ejecutar con `--verbosity normal` o en el explorador de pruebas del IDE.

---

## 12. Categorizar pruebas con [Trait]

Usar `[Trait]` para agrupar pruebas por categoría y ejecutar subconjuntos en CI o durante el desarrollo:

```csharp
[Fact]
[Trait("Categoria", "Validaciones")]
public void Validar_ConDatosValidos_DevuelveExito() { /* ... */ }

[Fact]
[Trait("Categoria", "BaseDeDatos")]
public void Guardar_OrdenNueva_PersisteCambios() { /* ... */ }

[Fact]
[Trait("Prioridad", "Alta")]
public void Procesar_OrdenCritica_NotificaInmediatamente() { /* ... */ }
```

### Ejecutar por categoría

```bash
# Solo pruebas de validaciones
dotnet test --filter "Categoria=Validaciones"

# Excluir pruebas lentas
dotnet test --filter "Categoria!=Lentas"

# Combinar filtros
dotnet test --filter "Categoria=Validaciones&Prioridad=Alta"
```

---

## 13. Fixtures compartidos en xUnit

### IClassFixture — compartir recursos dentro de una clase de pruebas

Cuando un recurso es costoso de crear (ej. conexión, configuración), compartirlo entre todas las pruebas de la clase:

```csharp
public class ConfiguracionFixture : IDisposable
{
    public IConfiguration Config { get; }

    public ConfiguracionFixture()
    {
        Config = new ConfigurationBuilder()
            .AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["LimiteMaximo"] = "1000",
                ["MonedaPorDefecto"] = "COP"
            })
            .Build();
    }

    public void Dispose() { }
}

public class PedidoServiceTests : IClassFixture<ConfiguracionFixture>
{
    private readonly ConfiguracionFixture _fixture;

    public PedidoServiceTests(ConfiguracionFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Procesar_ConLimiteConfigurado_RespetaElLimite()
    {
        var limite = _fixture.Config.GetValue<int>("LimiteMaximo");
        // ...
    }
}
```

### ICollectionFixture — compartir recursos entre múltiples clases

```csharp
[CollectionDefinition("BaseDeDatos")]
public class ColeccionBaseDeDatos : ICollectionFixture<DatabaseFixture> { }

[Collection("BaseDeDatos")]
public class PedidoRepositoryTests
{
    private readonly DatabaseFixture _db;
    public PedidoRepositoryTests(DatabaseFixture db) => _db = db;
}

[Collection("BaseDeDatos")]
public class ClienteRepositoryTests
{
    private readonly DatabaseFixture _db;
    public ClienteRepositoryTests(DatabaseFixture db) => _db = db;
}
```

### IAsyncLifetime — para setup/teardown asíncrono

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    public IDbConnection Conexion { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Conexion = new SqlConnection("...");
        await Conexion.OpenAsync();
    }

    public async Task DisposeAsync()
    {
        await Conexion.CloseAsync();
    }
}
```

---

## 14. Generación de datos de prueba

### Bogus — datos realistas

**Bogus** genera datos falsos pero realistas, útil para evitar datos genéricos como `"test"` o `"aaa"`:

```csharp
using Bogus;

var fakerPedido = new Faker<Pedido>()
    .RuleFor(p => p.Producto, f => f.Commerce.ProductName())
    .RuleFor(p => p.Cantidad, f => f.Random.Int(1, 100))
    .RuleFor(p => p.Precio, f => f.Finance.Amount(10, 5000))
    .RuleFor(p => p.ClienteId, f => f.Random.AlphaNumeric(10))
    .RuleFor(p => p.Direccion, f => f.Address.FullAddress());

[Fact]
public void Validar_ConPedidoAleatorioValido_DevuelveExito()
{
    var pedido = fakerPedido.Generate();

    var resultado = _validador.Validar(pedido);

    resultado.EsValido.Should().BeTrue();
}
```

### AutoFixture — creación automática de objetos

**AutoFixture** llena automáticamente todas las propiedades con valores válidos, sin necesidad de configurar cada campo:

```csharp
using AutoFixture;

[Fact]
public void Validar_ConPedidoAutoGenerado_NoLanzaExcepcion()
{
    var fixture = new Fixture();
    var pedido = fixture.Create<Pedido>();

    var act = () => _validador.Validar(pedido);

    act.Should().NotThrow();
}
```

```bash
dotnet add package Bogus
dotnet add package AutoFixture
```

---

## 15. Herramientas y configuración

### Paquetes NuGet recomendados

```xml
<ItemGroup>
  <!-- Framework de pruebas -->
  <PackageReference Include="xunit" Version="2.9.2" />
  <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />

  <!-- Mocking (elegir uno) -->
  <PackageReference Include="Moq" Version="4.20.72" />
  <!-- Alternativa: <PackageReference Include="NSubstitute" Version="5.3.0" /> -->

  <!-- Aserciones (elegir uno) -->
  <PackageReference Include="FluentAssertions" Version="6.12.2" />
  <!-- ⚠️ FluentAssertions 7.x requiere licencia comercial -->
  <!-- Alternativa gratuita: <PackageReference Include="Shouldly" Version="4.2.1" /> -->

  <!-- Cobertura -->
  <PackageReference Include="coverlet.collector" Version="6.0.2" />

  <!-- Datos de prueba (opcionales) -->
  <!-- <PackageReference Include="Bogus" Version="35.6.1" /> -->
  <!-- <PackageReference Include="AutoFixture" Version="4.18.1" /> -->

  <!-- TimeProvider para .NET 8+ -->
  <!-- <PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="9.0.0" /> -->
</ItemGroup>
```

### Medir cobertura con Coverlet

```bash
dotnet test tests/MiProyecto.Tests/MiProyecto.Tests.csproj \
  --collect:"XPlat Code Coverage" \
  --results-directory ./TestResults
```

### Integración en CI/CD

Ejecutar las pruebas como paso previo al despliegue en GitHub Actions, GitLab CI, Azure DevOps, etc.

```yaml
# Ejemplo GitHub Actions
- name: Run tests
  run: dotnet test --no-build --verbosity normal
```

---

## Referencias

- [Unit testing best practices - .NET (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
- [xUnit - Documentación oficial](https://xunit.net/)
- [Moq - Librería de mocking para .NET](https://github.com/moq/moq4)
- [NSubstitute - Alternativa a Moq](https://nsubstitute.github.io/)
- [FluentAssertions - Aserciones expresivas](https://fluentassertions.com/)
- [Shouldly - Alternativa gratuita de aserciones](https://docs.shouldly.org/)
- [Bogus - Datos de prueba falsos](https://github.com/bchavez/Bogus)
- [AutoFixture - Creación automática de objetos](https://github.com/AutoFixture/AutoFixture)
- [TimeProvider en .NET 8 (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.timeprovider)
- [Coverlet - Cobertura de código](https://github.com/coverlet-coverage/coverlet)
- [FIRST principles](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
