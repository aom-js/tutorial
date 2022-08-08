# Типовые формы входящих данных

Чаще всего используются следующие кейсы:

- добавление данных: метод POST, обязывает указать минимально необходимые обязательные данные
- замена: метод PUT, применяет в качестве значения документа те данные, которые были переданы
- редактирование: метод PATCH, позволяет изменить произвольно взятые значения, контролирует значения в обязательных полях

В общем случае методы добавления и замены совпадают друг с другом по требованию к данным, а метод редактирования
должен уметь отреагировать на противоречия в обязательных данных.

Для этого рассмотрим следующий пример.

Используем компонент `BodyBuilder`, который позволит генерировать контроллеры над схемами данных. В качестве схем
могут использованы произвольно взятые значения: как схемы моделей данных, так и кастомные структуры, которые
могут быть использованы в формах.

```ts
/**
 * Указанный контроллер позволяет создавать локальные middleware для обработки схемы данных,
 * сохраненной в качестве статичного значения класса
 *
 * в качестве значений ожидаются данные из `Body`
 * в качестве JSON значения генерирует схему для декоратора `@RequestBody`
 *
 * может быть совмещен с контроллерами, унаследованных от <DocumentRoute>,
 * чтобы передавать обработанное значение `body` в другой контекст
 * */
/* eslint-disable import/no-cycle */
import { C } from "ts-toolbelt";
import { Body, Next, RequestBody, Use } from "aom";
import { Controller, DelayRefStack } from "aom";
import { Middleware, NextFunction, Responses } from "aom";
import { RouteRef, ThisRef, This } from "aom";
import { BaseModel } from "common/schemas";
import { SafeData } from "common/api/safe-data";
import { OpenApiRequestBody } from "aom/lib/openapi/types";

import { WrongDataResponse } from "common/api";
import { DocumentRoute } from "common/routes";

const SafeOrigin = ThisRef(<T extends typeof BodyBuilderConstructor>({ schema }: T) =>
  SafeData(schema)
);
const SafeOriginRequestBody = ThisRef(<T extends typeof BodyBuilderConstructor>(constructor: T) => [
  RequestBody(constructor.toJSON()),
]);
@Controller()
export class BodyBuilderConstructor<T extends C.Class<any, any>, R = C.Instance<T>> {
  static schema;

  static description?;

  body: R;

  static toJSON(description?: string): OpenApiRequestBody {
    return {
      schema: this.schema,
      description: description || this.description,
    };
  }

  @Middleware()
  @Use(BodyBuilderConstructor.Body)
  static async Attach<T extends DocumentRoute<typeof BaseModel>>(
    @This() self: BodyBuilderConstructor<typeof BaseModel>,
    @This(RouteRef()) route: T,
    @Next() next: NextFunction
  ) {
    route.body = self.body;
    return next();
  }

  @Middleware()
  @DelayRefStack(SafeOriginRequestBody)
  @Responses(WrongDataResponse.toJSON())
  static async Body(
    @Body(SafeOrigin) body,
    @This() self: BodyBuilderConstructor<typeof BaseModel>,
    @Next() next: NextFunction
  ) {
    self.body = body;
    return next();
  }
}

export function BodyBuilder<T extends C.Class<any, any>>(schema: T, description?: string) {
  return class extends BodyBuilderConstructor<T> {
    static schema = schema;

    static description = description;
  };
}
```

Создадим контроллер над моделью данных, который подразумевает получение списка и добавление в него нового элемента,
а также обеспечивает контроль над единичным значением, полученного по идентификатору.

```ts
// schemas/Contents.ts
@ComponentSchema()
export class Contents extends EnabledModel {
  @prop({ required: true })
  @IsString()
  @IsNotEmpty()
  @Expose()
  @JSONSchema({ description: "Название" })
  title!: string;

  @prop()
  @IsString()
  @IsOptional()
  @Expose()
  @JSONSchema({ description: "Описание" })
  description?: string;

  @prop()
  @IsMongoId()
  @CheckExists(() => Categories)
  @IsOptional()
  @Expose()
  @JSONSchema({ description: "Описание" })
  categoryId?: string;
}
```

Примением эту схему к типовым случаям ожидаемых форм. Для некоторых преобразований используются "хакерские" функции,
которые корректно работают с точки зрения генерации данных и логики кода, однако не поддаются ожидаемому типированию,
что следует иметь ввиду в случаях применения этих данных в исходном коде.

```ts
// `api/module/init.ts`

const schema = schemas.Contents;

@ComponentSchema()
@NoJSONSchema()
export class AddBody extends PickSchema(schema, ["name", "description"]) {}

@ComponentSchema()
@NoJSONSchema()
export class PatchBody extends PartialSchema(schema) {}

@Controller()
export class PatchRequest extends BodyBuilder(PatchBody) {}

@Controller()
export class PutRequest extends BodyBuilder(schema) {}
```

При наличии таких файлов становится доступно их применение в контроллерах.

Допускается как исключительная проверка схем, так и обработка даных на уровне `middleware`.

```ts
// `api/module/index.ts`

@Controller()
export class Data extends controllers.Data.data() {
  @Get()
  @Summary("Поиск по данным")
  @Responses(Data.toJSON([]))
  static Index(@This() self: Data) {
    await self.getData();
    return self.data;
  }

  @Get()
  @Summary("Добавить данные")
  @RequestBody({ schema: AddBody })
  @Responses(Data.toJSON())
  static Add(@SafeBody(AddBody) body: AddBody, @This() self: Data) {
    const document = await self.model.create(body);
    return document;
  }
}
```

```ts
// `api/module/elem.ts`

@Controller()
export class Elem extends controllers.Data.document() {
  @Get()
  @Endpoint()
  @Summary("Значение данных")
  @Responses(Elem.toJSON([]))
  static Index(@This() self: Data) {
    return self.document;
  }

  @Put()
  @Summary("Заменить данные")
  @Use(PutRequest.Attach) // подключим значение непосредственно к body текущего документа
  @UseNext(Elem.Index)
  static Replace(@Next() next: NextFunction) {
    return next(this.SaveBody, this.Define);
  }

  @Patch()
  @Summary("Обновить данные")
  @Use(PatchRequest.Body) // подключим обработку ожидаемого значения
  @UseNext(Elem.Index)
  static Update(
    @This(PatchRequest) { body }: PatchRequest,
    @This() self: Elem,
    @Next() next: NextFunction
  ) {
    elem.body = body; // присвоим значение `body` и сохраним его
    return next(this.SaveBody, this.Define);
  }
}
```
