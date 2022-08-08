# Типовые ответы

В качестве типовых ответов могут быть использованы сообщения об ошибках, некие типичные структуры
и статистические данные.

Для этого подойдет конструкция, которая может быть использована в качестве аргумента в декораторе `@Err()`,
и которая сгенерирует типовую схему, подходящую для `OpenAPI`-документации.

```ts
// `common/api/error-message.ts`

import { ComponentSchema, OpenApiResponse } from "aom";
import { IsNumber, IsOptional, IsString } from "class-validator";
import { JSONSchema } from "class-validator-jsonschema";
import { targetConstructorToSchema } from "class-validator-jsonschema";
import { ERROR_MESSAGE } from "common/constants";

interface IErrorData {
  message: string;
  status: number;
  data?: any;
}

@JSONSchema({
  description: "Стандартный ответ об ошибке",
})
@ComponentSchema()
export class ErrorMessage extends Error implements IErrorData {
  static status = 500;

  static description = ERROR_MESSAGE;

  @IsNumber()
  @JSONSchema({
    description: "Код ошибки",
  })
  status: number;

  @IsString()
  @JSONSchema({
    description: "Сообщение об ошибке",
  })
  message: string;

  @IsOptional()
  @JSONSchema({
    type: "object",
    description: "Детали ошибки",
  })
  data?: any;

  constructor(message: string, status?, data?) {
    super(message);
    this.status = status || (<any>this.constructor).status;
    this.data = data;
  }

  toJSON(): IErrorData {
    return {
      message: this.message,
      status: this.status,
      data: this.data,
    };
  }

  static toJSON(description?: string): OpenApiResponse {
    return {
      status: this.status,
      schema: targetConstructorToSchema(this),
      description: description || this.description,
    };
  }
}
```

Таким образом становится доступно создание типовых сообщений об ошибках

```ts
export class NotFoundResponse extends ErrorMessage {
  static status = 404;

  static description = ERROR_NOT_FOUND;
}

export class ConflictResponse extends ErrorMessage {
  static status = 409;

  static description = ERROR_CONFLICT;
}

export class WrongDataResponse extends ErrorMessage {
  static status = 400;

  static description = ERROR_WRONG_DATA;
}

export class AccessDenyResponse extends ErrorMessage {
  static status = 403;

  static description = ERROR_ACCESS_DENY;
}

export class AuthRequiredResponse extends ErrorMessage {
  static status = 403;

  static description = ERROR_AUTH_REQUIRED;
}
```

Аналогичная конструкция применима для успешных сообщений с типовой структурой данных.

```ts
// `common/api/responses.ts`

import { OpenApiResponse } from "aom";
import { IsNumber, IsObject, IsOptional, IsString } from "class-validator";
import { JSONSchema } from "class-validator-jsonschema";

export class CommonResponse {
  static status = 200;

  static description: string;

  static toJSON(description?: string | string[]): OpenApiResponse {
    return {
      status: this.status,
      schema: this,
      isArray: Array.isArray(description),
      description: description ? description.toString() : "",
    };
  }
}

export class MessageResponse extends CommonResponse {
  static description = "Типовое сообщение";

  @IsString()
  @IsOptional()
  @JSONSchema({ description: "Сообщение" })
  message?: string;

  constructor(message?: string) {
    super();
    if (message) this.message = message;
  }
}

export class DataResponse extends MessageResponse {
  static description = "Сообщение с данными";

  @IsObject()
  @IsOptional()
  @JSONSchema({ description: "Данные" })
  data?: any;

  constructor(message?: string, data?: any) {
    super(message);
    if (data) this.data = data;
  }
}

export class TotalDataResponse extends DataResponse {
  @IsNumber()
  @IsOptional()
  @JSONSchema({ description: "Общее количество данных" })
  total?: number;
}
```

Таким образом, что становится доступно их применение в качестве ответов, который
генерирует данный участок маршрута.

```ts
// возможна ошибка проверки данных на декораторе `@SafeQuery`, о чем важно сообщить документации
export class Items extends controllers.Items.data() {
  @Middleware()
  @QueryParameters(...ItemsSearch.toJSON())
  @Responses(WrongDataResponse.toJSON(ERROR_QUERY_SEARCH))
  static async Search(
    @This() self: Items,
    @SafeQuery(ItemsSearch) query: ItemsSearch,
    @Next() next: NextFunction
  ) {
    Object.assign(self.where, { ...query });
    return next();
  }
}
```

```ts
@Bridge("/", LoginRoute)
@Controller()
class Root extends RootRoute {
  routes = $root.routes;

  docs = Docs;

  url = url;

  @Post("/logout")
  @Use(Auth.Required)
  @Summary("Завершить сессию")
  @Responses(MessageResponse.toJSON())
  static async Logout(@This(Auth) auth: Auth) {
    //
    const { _id: tokenId } = auth.token;
    await models.UsersLoginsTokens.updateOne({ _id: tokenId }, { $set: { enabled: false } });

    return new MessageResponse("Сессия завершена");
  }
}
```

Все возможные варианты `Responses` будут объединены в единый список на уровне всех `middleware` и
даст полную картинку возможных состояний, в которых может в произвольно взятый момент находиться этот
`endpoint`.
