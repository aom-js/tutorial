# Пример получения и обработки значения по `_id`

Типовым случаем обращения к данным является получение значения документа по `_id`
При этом подразумевается, что могут быть использованы особые расширенные логики, которые обогатят информацию
о найденном документе.

Для реализации данной задачи используется паттерн "Фабрика", описанный ранее, для которого применяется собственная
логика общего класса.

```ts
// common/data-elem.ts
/**
 * Данный класс создает контроллер `DocumentRoute`, который может быть унаследован другими контроллерами
 * и позволяет быстро и безопасно извлечь значение из модели данных по его `_id`.
 * В общем случае данный контроллер обеспечивает возможность быстро включить в контекст документ,
 * который был найден по `_id`, а также обеспечить быстрое сохранение промежуточно накопленных данных.
 *
 */
import { C } from "ts-toolbelt";
import { DocumentType, getModelForClass, ReturnModelType } from "@typegoose/typegoose";
import { BaseModel } from "schemas/common/base";
import { ErrorMessage, NotFoundResponse, MongoIDParam, MongoIDSchema } from "common/api";
import { NextFunction, Params, Controller, Middleware, Next, ErrorFunction, Err } from "aom";
import { PathParameters, DelayRefStack, This, ThisRef, Use } from "aom";
import { Types } from "mongoose";
import { OpenApiPathParameters } from "aom/lib/openapi/types";

const RefParams = ThisRef(<T extends typeof DocumentRoute>(constructor: T) => [
  PathParameters(constructor.param()),
]);

@Controller()
@Use(DocumentRoute.Prepare, DocumentRoute.Define)
export class DocumentRoute<T extends typeof BaseModel, U = C.Instance<T>> {
  static schema;

  static param_id: string;

  notFoundMessage: string = "Not found";

  _id: Types.ObjectId;

  model!: ReturnModelType<T>;

  document: DocumentType<U>;

  data: T extends U;

  body: Partial<U> = {};

  static async parseParamsId(params): Promise<Types.ObjectId> {
    return new Types.ObjectId(params[this.param_id]);
  }

  @Middleware()
  @DelayRefStack(RefParams)
  @Responses()
  static async Prepare(
    @This() elem: DocumentRoute<typeof BaseModel>,
    @Params() params: CustomData,
    @Err(ErrorMessage) err: ErrorFunction,
    @Next() next: NextFunction
  ): Promise<ErrorResponse<ReturnType<NextFunction>>> {
    try {
      elem._id = await this.parseParamsId(params);
      return next();
    } catch (e) {
      return err(e);
    }
  }

  @Middleware()
  @Responses(NotFoundError.toJSON())
  static async Define(
    @This() elem: DocumentRoute,
    @Err(NotFoundError) err: ErrorFunction,
    @Next() next: NextFunction
  ) {
    elem.document = await elem.model.findById(elem._id);
    if (!elem.document) {
      return err(elem.notFoundMessage);
    }
    return next();
  }

  @Middleware()
  static async SaveBody(@This() elem: DocumentRoute, @Next() next: NextFunction) {
    await elem.model.updateOne({ _id }, { $set: { ...elem.body } })
    return next();
  }

  static toString(): string {
    return MongoIDParam(this.param_id);
  }

  static param(): OpenApiPathParameters {
    return {
      [this.toString()]: {
        name: this.param_id,
        description: "Идентификатор",
        schema: MongoIDSchema,
      },
    };
  }


  /* JSON-значение данного класса - это схема успешного ответа, которое принимает `@Responses()`
    интерпретирует ответ в зависимости от указания того, список это или
  */
  static toJSON(
    description: string = "документ данных"
  ): OpenApiResponse {
    return {
      status: 200,
      schema: this.schema,
      description,
    };
  }
}

/**
 * Фабрика возвращает класс с установленными значениями схемы и созданной моделью данных.
 */

export function CreateDocumentRoute<T extends typeof BaseModel>(schema: T) {
  return class extends DocumentRoute<T> {
    static schema = schema;

    model = getModelForClass(schema);
  };
}
```

Позволяет создавать типовые контроллеры, которые предоставляют доступ к документу, извлекая значение из
адресной строки. В дальнейшем над извлеченными значениями допускаются расширенные действия: удаление,
изменение, специфические операции.

```ts
// contents/content.ts
import { Controller, This, Get, Use, Responses, Middleware } from 'aom';
import { SafeBody, CreateDocumentRoute, QueryString } from "common"; // используем типовые обработчики данных
import { Contents as ContentsSchema } from 'schemas`
import { CreateDataRoute } from "../data-route"; // используем фабрику

@Controller()
export class Content extends CreateDocumentRoute(ContentsSchema) {
  @Middleware()
  static async Init(@This() self: Content, @Next() next: NextFunction) {
    self.where = { enabled: true };
    self.populates.add({path: "user"});
    return next();
  }

  @Get()
  @Endpoint()
  @Summary('Элемент контента')
  @Use(Content.Init)
  @Responses(Contents.toJSON())
  static async Index(@This() content: Content) {
   return content.document;
  }

  @Patch()
  @Summary("Изменить контент")
  @Responses(Contents.toJSON("Новый документ"))
  @RequestBody({schema: Content.schema})
  static async Add(@This() self: Content, @SafeBody(Content.schema) body: Content.schema) {
    return self.model.create(body);
  }
}

```
