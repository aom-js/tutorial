# Контроллеры из схем данных

Наиболее частым случаем использования роутеров является создание элементов справочников данных,
содержащих типовые конструкции: получение списка из модели, добавление, удаление, редактирование.

Рассмотрим несколько способов, позволяющих решить такую задачу. Приведенные примеры не ограничивают
функциональность, доступную для применения на фреймворке, однако дают базовое понимание возможностей
и позволят расширить свою функциональность на смежные задачи.

## #1 Фабрикация

Используем паттерн "Фабрика" для создания типового класса, который будет содержать в себе требуемый
набор свойств, и позволит производить базовые операции над моделью данных, переданной как аргумент.

```ts
/**
 * рассмотрим пример, в котором создается контроллер, который позволяет производить следующие
 * базовые действия над входящей моделью.
 *
 * Данный класс - это контроллер `aom`, и будет корректно унаследован своими потомками, используемыми
 * как контролер `aom`.
 *
 * Таким образом, что контекст конструкции `@This()` в аргументах роутера позволит получить объект,
 * который умеет формировать аргументы к будущему запросу данных к модели, которая создается из переданной схемы
 *
 * Подразумевается, что схемами данных являются документы `MongoDB`, описанные в нотации `Typegoose`.
 * Главная цель такого класса хранить безопасные значения, которые можно надежно использовать в построении
 * типового запроса к базе.
 *
 * В данном случае предоставляется возможность создать класс, который можно быстро обогащать значениями
 * из http-контента, а конкретно использовать значения `?query=string` для того, чтобы безопасно их
 * интепретировать.
 *
 * В общем случае данный класс затем совмещается с типовыми классами, которые дают middleware-преобразования,
 * и финализируют рендеринг значений.
 *
 * Таким образом, существующий класс создает свойство `getData`, которое сохраняет результат запроса, построенного
 * на значениях свойств `where`, `populates`, `sortOrder` и `pager`.
 *
 * Значения `data` считается полным на момент обращения к нему и может быть в произвольный момент пересчитано.
 * Таким образом становится доступна возможность совмещать контроллеры друг с другом, обеспечивая получение типовых
 * структур данных, используя ожидаемую типовую логику.
 *
 * Примеры будут приведены дальше по тексту.
 */
// common/data-route.ts
import { Controller, OpenApiResponse } from "aom";
import { C } from "ts-toolbelt";
import { DocumentType, getModelForClass, ReturnModelType } from "@typegoose/typegoose";
import { FilterQuery, PopulateOptions } from "mongoose";
import { BaseModel } from "schemas/common/base";
import { PagerFilter } from "api/init/classes/query-filter";

@Controller()
export abstract class DataRoute<T extends typeof BaseModel, U = C.Instance<T>> {
  static schema;

  model!: ReturnModelType<T>;

  data: DocumentType<U>[] = [];

  where: FilterQuery<U> = {};

  sortOrder: { [K in keyof U]?: 1 | -1 };

  pager: PagerFilter; // используется схема `?offset=number&limit=number`

  populates: Set<string | PopulateOptions> = new Set();

  /** стандартное извлечение данных на основании используемых ограничений */
  async getData(): Promise<void> {
    const query = this.model.find(<any>this.where);
    // добавим постраничную навигацию
    if (this.pager) {
      const { offset, limit } = this.pager;
      query.skip(offset).limit(limit);
    }
    // добавим сортировку
    if (this.sortOrder) {
      query.sort(this.sortOrder);
    }
    // раскроем populate конструкции
    this.populates.forEach((populate) => query.populate(populate as any));
    // извлечем данные без преобразования в документы
    this.data = await query.lean();
  }

  /* JSON-значение данного класса - это схема успешного ответа, которое принимает `@Responses()` 
    интерпретирует ответ в зависимости от указания того, список это или 
  */
  static toJSON(
    description: string | string[] = "типовая структура данных по схеме"
  ): OpenApiResponse {
    return {
      status: 200,
      schema: this.schema,
      isArray: Array.isArray(description),
      description: description.toString(),
    };
  }
}
/**
 * Фабрика роутера: позволяет передать на вход схему, и вернуть класс с созданной моделью.
 */

export function CreateDataRoute<T extends typeof BaseModel>(schema: T) {
  return class extends DataRoute<T> {
    static schema = schema;

    model = getModelForClass(schema);
  };
}

export default CreateDataRoute;
```

Данный класс обеспечит возможность создания типовых контроллеров, которые позволят быстро и безопасно
обращаться к собственным данным, используя контекст http-запросов.

```ts
/**
 * Данный класс создает контроллер, который доступен при обращении к нему по методу `GET` и `POST`.
 * Позволяет получить список данных и добавить в него новое значение.
 *
 * В общем случае может быть пристыкован к любому `@Bridge`, и обеспечит доступ к данным по указанному
 * адресу.
 *
 * Хорошо совмещается с типовыми библиотеками обработки входящих данных (будут рассмотрены в соответствующем разделе).
 *
 * Позволяет быстро создавать конструкции вида `/content/id_{id}/...`, `/content/stats` и так далее.
 *
 * Могут быть использованы в других контроллерах, чтобы получить типовые данные с уточненным контекстом.
 */
// contents/index.ts
import { Controller, This, Get, Use, Responses, Middleware } from 'aom';
import { SafeBody, CreateDataRoute, QueryString } from "common"; // используем типовые обработчики данных
import { Contents as ContentsSchema } from 'schemas`
import { CreateDataRoute } from "../data-route"; // используем фабрику

@Controller()
export class Contents extends CreateDataRoute(ContentsSchema) {
  @Middleware()
  static async Init(@This() self: Content, @Next() next: NextFunction) {
    self.where = { enabled: true };
    self.populates.add({path: "user"});
    return next();
  }

  @Get()
  @Summary('Получить контент')
  @Use(Content.Init, QueryString.Pager)
  @Responses(Contents.toJSON(["Список контента"]))
  static async Index(@This() content: Content) {
   await content.getData();
   return content.data;
  }

  @Post()
  @Summary("Добавить контент")
  @Responses(Contents.toJSON("Новый документ"))
  @RequestBody({schema: Content.schema})
  static async Add(@This() self: Content, @SafeBody(Content.schema) body: Content.schema) {
    return self.model.create(body);
  }
}
```

Соответственно, структуры классов контроллеров могут быть произвольно сложными, и совмещать в себе другие действия.
Будут рассмотрены примеры, которые позволят интегрировать такие контроллеры друг с другом, и создавать сложные схемы
ответов, проверять полномочия и иначе использовать контекст общих данных.

Рассматриваются следующие примеры:

## Получение доступа к документу по `_id`

Пример, представленный в документе [get-by-id](./get-by-id.md) позволяет создавать типовой класс, который
может быть подключен через `@Bridge`, и использоваться для расширения функционала контроллера.

```ts
import { Contents as ContentsSchema } from "schemas";
import { Content } from "./content";

@Controller()
@Bridge(`/content_${Content}`, Content) // подключили типовой обработчик единицы контента
export class Contents extends CreateDataRoute(ContentsSchema) {
  // ... Init, Index, Add
}
```

## Обобщение контроллеров

Пример, представленный в документе [common-controllers](./common-controllers.md) позволяет создавать конструкцию
генерации типовых набор контроллеров вокруг произвольной схемы данных.

```ts
import { Contents as ContentsSchema } from "schemas";
import { controllers } from "common/models";

@Controller()
@Bridge(`/content_${Content}`, Content)
export class Contents extends controllers.Contents.data() {
  // ... Init, Index, Add ...
}

@Controller() 
export class Content extends controllers.Contents.document() {
  // ... Index, Patch, Put, SaveBody...
}
```

## Получение доступа к документу по `_id`

Пример, представленный в документе [get-by-id](./get-by-id.md) позволяет создавать типовой класс, который
может быть подключен через `@Bridge`, и использоваться для расширения функционала контроллера.

```ts
import { Contents as ContentsSchema } from "schemas";
import { Content } from "./content";

@Controller()
@Bridge(`/content_${Content}`, Content) // подключили типовой обработчик единицы контента
export class Contents extends CreateDataRoute(ContentsSchema) {
  // ... Init, Index, Add
}
```

## Расширенный рендер значений

Пример, представленный в документе [extended-render](./extended-render.md) позволяет расширять информацию о
возвращаемой структуре данных.

```ts
import { Contents as ContentsSchema } from "schemas";
import { Renders } from "common";

@Controller()
@Bridge(`/content_${Content}`, Content) // подключили типовой обработчик единицы контента
export class Contents extends CreateDataRoute(ContentsSchema) {
  @Get()
  @Summary("Данные со списком контента")
  @UseNext(Renders.CountData)
  static async Index(@This() self: Content, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }
}
```
