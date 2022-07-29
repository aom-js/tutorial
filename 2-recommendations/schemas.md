# Структура схем и моделей

Схемы данных допускают достаточно сильную связность между собой. В общем случае при корректной
их организации становится доступна возможность создания типовых моделей, в том числе обогащенных
составной логикой.

Структура файлов со схемами чаще всего будет сводиться к следующему виду:

```
-+ schemas/
--+ Items/
---- index.ts
---- Items.ts
---- Categories.ts
---- Brands.ts
--+ Users/
---- index.ts
---- Users.ts
---- Logins.ts
---- Tokens.ts
--| index.ts
```

```ts
// schemas/index.ts
export * from "./Users";
export * from "./Items";
```

```ts
// schemas/Items/index.ts
export * from "./Items";
export * from "./Categories";
export * from "./Brands";
```

```ts
// schemas/Users/index.ts
export * from "./Users";
export * from "./Logins";
export * from "./Tokens";
```

Значения файлов в общем случае содержит в себе `typegoose`-схемы, который в том числе могут
опираться друг на друга для получения виртуальных значений.

```ts
// `schemas/Users/Users.ts`
import { UsersLogins } from "./Logins";

@ComponentSchema()
export class Users extends EnabledModel {
  @prop()
  @IsString()
  @Expose()
  name: string;

  @prop({
    ref: () => UsersLogins,
    localField: "_id",
    foreignField: "userId",
    match: { type: LoginsTypes.PLAIN },
    justOne: true,
  })
  @IsOptional()
  @ValidateNested()
  @Type(() => UsersLogins)
  @JSONSchema({ description: "Логин", readOnly: true })
  readonly login?: UsersLogins;

  @prop({
    ref: () => UsersLogins,
    localField: "_id",
    foreignField: "userId",
    match: { type: LoginsTypes.EMAIL },
    justOne: true,
  })
  @IsOptional()
  @ValidateNested()
  @Type(() => UsersLogins)
  @JSONSchema({ description: "Email", readOnly: true })
  readonly email?: UsersLogins;
}
```

```ts
// `schemas/Users/Logins.ts`
import { Users } from "./Users";

@ComponentSchema()
export class UsersLogins extends LoginsModel {
  @prop({ ref: () => Users, index: true })
  @IsMongoId()
  @Exclude()
  userId: Types.ObjectId;
}
```

При корректной организации схем данных возможно их преобразование в модели

```ts
// common/models.ts
import { getModelForClass, ReturnModelType } from "@typegoose/typegoose";
export * as schemas from "schemas";

function buildModels<
  K extends keyof typeof schemas,
  S extends typeof schemas,
  T = { [N in K]: ReturnModelType<S[N]> }
>(schemas: S): T {
  const result = {};
  Object.keys(schemas).forEach((key) => {
    const schema = schemas[key];
    Object.assign(result, { [key]: getModelForClass(schema) });
  });
  return result as T;
}

export default buildModels(schemas);
```

Таким образом становится доступно их использование в произвольно взятом контексте

```ts
// api/routes/route.ts
import * as schemas from "schemas";
import models from "common/models";

@Controller()
export class Route extends SchemaRoute(schemas.DifficultData) {
  static async Data(@This() { document }: Route) {
    const result = new DataResponse();
    const { attrId } = document;
    result.document = document;
    result.data = await models.RelatedData.find({ attrId });
    return result;
  }
}
```

```ts
// schemas/Users/Users.ts
import models from "common/models";

@Controller()
export class Users extends EnabledModel {
  @prop()
  @IsString()
  @Expose()
  name: string;

  async getLogins(this: DocumentType<Users>): UsersLogins[] {
    return models.UsersLogins.find({ userId: this._id });
  }
}
```
