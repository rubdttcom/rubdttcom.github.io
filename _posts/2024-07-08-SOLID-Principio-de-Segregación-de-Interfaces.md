La abstracción oculta la implementación en el diseño orientado a objetos. Se logra con clases abstractas e interfaces. Aquí se detalla el Principio de Segregación de Interfaces en SOLID.

#### Tabla de Contenidos
1. ¿Qué es una Interfaz?
2. ¿Qué es el Principio de Segregación de Interfaces?
3. Razones para Seguir el Principio de Segregación de Interfaces
4. CodeSmells por Violaciones del ISP y Cómo Arreglarlos
    - Una Interfaz Voluminosa
    - Dependencias No Usadas
    - Métodos que Lanzan Excepciones
    - Refactorización de CodeSmells
5. Entonces, ¿Deberían las Interfaces Tener Siempre un Solo Método?
6. El Principio de Segregación de Interfaces y Otros Principios SOLID


### ¿Qué es una Interfaz?


Una Interfaz es un conjunto de abstracciones que una clase implementadora debe seguir. Definimos el comportamiento pero no lo implementamos:

```
interface Dog {
  void bark();
}
```

Tomando la interfaz como una plantilla, podemos luego implementar el comportamiento:

```java
class Poodle implements Dog {
  public void bark(){
    // Implementación específica del poodle    
  }
}
```

### ¿Qué es el Principio de Segregación de Interfaces?
El Principio de Segregación de Interfaces (ISP) establece que un cliente no debe estar expuesto a métodos que no necesita. Declarar métodos en una interfaz que el cliente no necesita, contamina la interfaz y conduce a una interfaz "voluminosa" o "gorda".

#### Razones para Seguir el Principio de Segregación de Interfaces
Veamos un ejemplo:

Un cliente puede pedir una hamburguesa, patatas fritas o una combinación de ambas:

```java
interface OrderService {
    void orderBurger(int quantity);
    void orderFries(int fries);
    void orderCombo(int quantity, int fries);
}
```

Dado que un cliente puede pedir patatas fritas, una hamburguesa o ambas, decidimos poner todos los métodos de pedido en una sola interfaz.

Ahora, para implementar un pedido solo de hamburguesas, **nos vemos obligados** a lanzar una excepción en el método `orderFries()`:

```java
class BurgerOrderService implements OrderService {
    @Override
    public void orderBurger(int quantity) {
        System.out.println("Recibido pedido de " + quantity + " hamburguesas");
    }

    @Override
    public void orderFries(int fries) {
        throw new UnsupportedOperationException("No hay patatas en el pedido solo de hamburguesas");
    }

    @Override
    public void orderCombo(int quantity, int fries) {
        throw new UnsupportedOperationException("No hay combo en el pedido solo de hamburguesas");
    }
}
```

De manera similar, para un pedido solo de patatas fritas, también tendríamos que lanzar una excepción en el método `orderBurger()`.

Y este no es el único inconveniente de este diseño. Las clases `BurgerOrderService` y `FriesOrderService` también tendrán efectos secundarios no deseados cada vez que hagamos cambios en nuestra abstracción.

Supongamos que decidimos aceptar un pedido de patatas en unidades como libras o gramos. En ese caso, lo más probable es que tengamos que añadir un parámetro de unidad en `orderFries()`. ¡Este cambio también afectará a `BurgerOrderService` aunque no esté implementando este método!

Al violar el ISP, enfrentamos los siguientes problemas en nuestro código:

1. Los desarrolladores se confunden con los métodos que no necesitan.
2. El mantenimiento se vuelve más difícil: un cambio en una interfaz nos obliga a cambiar clases que no implementan la interfaz.
3. Violar el ISP también conduce a la violación de otros principios como el Principio de Responsabilidad Única.

#### CodeSmells por Violaciones del ISP y Cómo Arreglarlos
Algunos CodeSmells que podrían indicar una violación del ISP.

##### Una Interfaz Voluminosa
En interfaces voluminosas, muchas operaciones no se usan. El ISP sugiere que deberíamos necesitar la mayoría de los métodos. Probar estas interfaces implica configurar dependencias complicadas.

##### Dependencias No Usadas
Pasar null o un valor equivalente a un método indica una violación del ISP. Por ejemplo, pasar cero en orderCombo() para un pedido solo de hamburguesas. Deberíamos usar una interfaz separada para patatas.

##### Métodos que Lanzan Excepciones
Como en nuestro ejemplo de hamburguesas, si encontramos una `UnsupportedOperationException`, una `NotImplementedException`, o excepciones similares, huele a un problema de diseño relacionado con el ISP.

#### Refactorización de CodeSmells
Podemos refactorizar nuestro código del pedido de hamburguesas para tener interfaces separadas para `BurgerOrderService` y `FriesOrderService`:

```java
interface BurgerOrderService {
    void orderBurger(int quantity);
}

interface FriesOrderService {
    void orderFries(int fries);
}
```

En caso de que tengamos una dependencia externa, podemos usar el patrón Adaptador para abstraer los métodos no deseados, lo que hace que dos interfaces incompatibles sean compatibles usando una clase adaptadora.

Por ejemplo, supongamos que `OrderService` es una dependencia externa que no podemos modificar y necesitamos usar para hacer un pedido. Usaremos el Patrón Adaptador de Objetos para adaptar `OrderService` a nuestra interfaz objetivo, es decir, `BurgerOrderService`. Para esto, crearemos la clase `OrderServiceObjectAdapter` que contiene una referencia al `OrderService` externo.

```java
interface IOrderService {
    void orderBurger(int quantity);
    void orderFries(int fries);
    void orderCombo(int quantity, int fries);
}

class OrderServiceObjectAdapter implements BurgerOrderService {
    private IOrderService _orderServiceAdaptee;
    public OrderServiceObjectAdapter(IOrderService orderServiceAdaptee) {
        super();
        _orderServiceAdaptee = orderServiceAdaptee;
    }

    @Override
    public void orderBurger(int quantity) {
        _orderServiceAdaptee.orderBurger(quantity);
    }
}
```

Ahora, cuando un cliente quiera usar `BurgerOrderService`, podemos usar `OrderServiceObjectAdapter` para envolver la dependencia externa:

```java
class ComboOrderService implements IOrderService {
    @Override
    public void orderBurger(int quantity) {
        System.out.println("Combo order: " + quantity + " burgers");
    }

    @Override
    public void orderFries(int fries) {
        System.out.println("Combo order: " + fries + " fries");
    }

    @Override
    public void orderCombo(int quantity, int fries) {
        System.out.println("Combo order: " + quantity + " burgers and " + fries + " fries");
    }
}

interface BurgerOrderService {
    void orderBurger(int quantity);
}


class Main{
    public static void main(String[] args){
        IOrderService orderService = new ComboOrderService();
        BurgerOrderService burgerService = new OrderServiceObjectAdapter(new ComboOrderService());
        burgerService.orderBurger(4);
    }
}
```

Seguimos usando los métodos de `IOrderService` y el cliente solo depende de orderBurger(). Utilizamos `IOrderService` como dependencia externa y hemos reestructurado el código para evitar luna violación del ISP.

### Entonces, ¿Deberían las Interfaces Tener Siempre un Solo Método?
Aplicar el ISP al extremo resultará en interfaces de un solo método, también conocidas como __interfaces de rol__.

Esta solución resolverá el problema de la violación del ISP. Aún así, puede resultar en una violación de la cohesión en las interfaces, lo que resulta en una base de código dispersa que es difícil de mantener. 

### El Principio de Segregación de Interfaces y Otros Principios SOLID
El ISP está particularmente asociado con el Principio de Sustitución de Liskov (LSP) y el Principio de Responsabilidad Única (SRP).

En nuestro ejemplo, hemos lanzado una `UnsupportedOperationException` en `BurgerOrderService`, lo cual es una violación del LSP ya que el hijo no está extendiendo realmente la funcionalidad del padre sino restringiéndola.

El SRP establece que una clase debe tener solo una razón para cambiar. Si violamos el ISP y definimos métodos no relacionados en la interfaz, la interfaz tendrá múltiples razones para cambiar - una por cada uno de los clientes no relacionados que necesitan cambiar.

Otra relación interesante del ISP es con el Principio de Abierto/Cerrado (OCP), que establece que una clase debe estar abierta para extensión pero cerrada para modificación. En nuestro ejemplo, tenemos que modificar `IOrderService` para añadir otro tipo  de pedido. Si hubiéramos implementado `IOrderService` para tomar un objeto `Order` genérico como parámetro, nos habríamos ahorrado una potencial violación del OCP y habríamos resuelto la violación del ISP:

```java
interface OrderService {
    void submitOrder(Order order);
}
```

[Patrón Adaptador](https://es.wikipedia.org/wiki/Adaptador_(patr%C3%B3n_de_dise%C3%B1o))

---

Inspirado en [Interface Segregation Principle: Everything You Need to Know](https://reflectoring.io/interface-segregation-principle/)
