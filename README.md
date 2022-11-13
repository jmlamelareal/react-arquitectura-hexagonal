# React Arquitectura Hexagonal

## Separación por capas

Las capas definen directorios, la arquitectura por capas define como interactuan los elementos de una capa con elementos de otra, incluso si pueden hacerlo.

### Capa de dominio

En esta capa se encuentran nuestros modelos, la lógica de negocio, validaciones, gestión de errores, etc. Esta capa debería explicar de que va la aplicación, es la más pura.

### Capa de aplicación

En esta capa tendremos la gestión de roles y permisos, gestión de estado y la más importante que son los casos de uso.

### Casos de uso

Los casos de uso son la interacción entre el usuario y la aplicación, es decir, las acciones que puede realizar el usuario. Estos casos de uso se componen de una serie de pasos que se ejecutan en orden, estos pasos pueden ser llamadas a otros casos de uso, llamadas a servicios externos, etc.

### Capa de delivery

Esta capa es el mecanismo de entrega, la que expone los casos de uso.

### Capa de infraestructura

Esta capa es la que se encarga de la persistencia de datos, la comunicación con servicios externos, etc. Es la capa donde sabes que el código está acoplado a la tecnología que estás usando.

## Conocimiento entre capas

El conocimiento entre capas va de fuera hacia dentro, es decir, las capas más excéntricas conocen las más internas, pero las más internas no conocen las más externas. Por ejemplo, delibery podrá hacer uso de las capas de aplicación y de dominio. En cambio la capa de dominio no podrá hacer uso de las capas de delivery, infraestructura o aplicación.

## Dominio

Se recomienda definir las entiedades con interfaces.

```typescript
import { Id } from "../../../core/types/id";

export interface Achievement {
  id: Id;
  name: string;
  date: Date;
}
```

### Repositorios (mal llamados servicios)

Son piezas que interactúan con datos. Encuentran entidades de dominio, actualizan, borran, etcétera. Debemos definir una interfaz en el dominio la cual debe ser implementada en la capa de infraestructura.

- En el backend podría ser la pieza que interactua con al base de datos.
- En el frontend podría ser la pieza que interactua con el localstorage.
- En el frontend podría ser la pieza que interactua con un backend mediante peticiones http.

```typescript
import { Achievement } from "./achievement";

export interface AchievementRepository {
  findAll(): Promise<Achievement[]>;
  findById(id: string): Promise<Achievement>;
  save(achievement: Achievement): Promise<void>;
  delete(id: string): Promise<void>;
}
```

### Infraestructura

El cómo se interactúan con los datos (repositorios) es un detalle de implentación, en esta capa se implementan los repositorios.

```typescript
import { AchievementRepository } from "../domain/achievement/achievement-repository";
import { Achievement } from "../domain/achievement/achievement";

export class AchievementHttpRepository implements AchievementRepository {
  async findAll(): Promise<Achievement[]> {
    const response = await fetch("http://localhost:3000/achievements");
    const achievements = await response.json();
    return achievements;
  }
}
```

```typescript
import { AchievementRepository } from "../domain/achievement/achievement-repository";
import { Achievement } from "../domain/achievement/achievement";

export class ArchievementLocalStorageRepository
  implements AchievementRepository
{
  async findAll(): Promise<Achievement[]> {
    const achievements = localStorage.getItem("achievements");
    return achievements;
  }
}
```

#### DTOs (Data Transfer Object)

Cuando necesitamos transformar datos entre capas (para normalizar nombres, simplificar un objeto, transformación de datos por ejemplo de string a objeto Date), usaremos los DTOs que estarán defindios en la capa de infraestructura.

```typescript
import { Id } from "../../../core/types/id";

export class AchievementDto {
  id: Id;
  name: string;
  date: string;
}
```

```typescript
import { AchievementRepository } from "../domain/achievement/achievement-repository";
import { Achievement } from "../domain/achievement/achievement";

export class AchievementHttpRepository implements AchievementRepository {
  async findAll(): Promise<Achievement[]> {
    const response = await fetch("http://localhost:3000/achievements");
    // [{id: "1", name: "Achievement 1", date: "2020-01-01"}]
    const achievementsDtos: AchievementDto = await response.json();
    return achievementsDtos.map((x) => {
      return {
        ...x,
        date: new Date(x.date),
      };
    });
  }
}
```

#### Mappers o Converters

Transforman un objeto a otro. Podemos usarlos en los repositorios para transformar los datos que vienen de la capa de infraestructura a la capa de dominio.

```typescript
import { AchievementRepository } from "../domain/achievement/achievement-repository";
import { Achievement } from "../domain/achievement/achievement";
import { AchievementDto } from "./achievement-dto";

export class AchievementHttpRepository implements AchievementRepository {
  async findAll(): Promise<Achievement[]> {
    const response = await fetch("http://localhost:3000/achievements");
    const achievementDto: AchievementDto[] = await response.json();
    return achievementDto.map((achievement) => ({
      id: achievement.id,
      name: achievement.name,
      date: new Date(achievement.date),
    }));
  }
}
```

### Aplicación

#### Casos de uso

Los casos de uso van a usar los repositorios para orquestar las acciones que se van a realiar sobre la aplicación. Ellos sabe a que servicios de dominio llamar, a que repositorios llamar, etc. Van a preparar todo y empaquetarlo de tal forma que se lo den ya mascado al consumidor de dicho caso, por ejemplo, los componentes de React.

A los casos de usos les pasamos la interface del repositorio ya que así podremos pasarles diferentes implementaciones (http, localStorage, etc) y así poder testearlos con mayor facilidad, en definitiva polimorfismo.

```typescript
import { Achievement } from "../domain/achievement/achievement";
import { AchievementRepository } from "../domain/achievement/achievement-repository";

export class GetAllAchievementsUseCase {
  constructor(private readonly achievementRepository: AchievementRepository) {}

  execute(): Promise<Achievement[]> {
    return this.achievementRepository.findAll();
  }
}
```

##### IoC (Inversion of control) Dependency inversion principle

El principio de inversión de dependencias nos dice que las dependencias de una clase deben ser inyectadas desde fuera. Esto nos permite testear las clases de forma aislada, ya que no dependen de nada externo.

## Uniendo todo

- En la capa de dominio tendremos las definiciones de:

  - Entidades
  - Servicios (operaciones sin estado referentes al dominio)
  - Repositorios

- En la capa de aplicación tendremos los casos de uso: Estos casos de uso van a usar las entidades, servicios y repositorios para orquestar las acciones que se van a realiar sobre la aplicación.

- En la capa de infraestructura tendremos las implementaciones de:

  - Repositorios
  - DTOs
  - Mappers

Ejemplo: Desde nuestra aplicación React (Infaestructura) vamos a llamar a un caso de uso (Aplicación) que va a usar un repositorio (Infraestructura) para obtener los datos de la API (Infraestructura).
