# Построение сложных запросов

Наиболее частый случай использование механик, описанных в текущем файле - это поиск по
списку значений или комбинирование поиска по контекстным моделям данных. Нередко требуется
использовать контекст запроса, который позволит, например, использовать данные текущего
пользователя или другие особые значения.

Решение задачи будет достигаться за счет создания контроллера, который будет использовать
переданную ему структуру в качестве источника значений, и будет совмещен с контекстом класса
`DataRoute` для быстрой передачи через `middleware`.

```ts
// `common/classes/query-builder.ts`
import { C } from "ts-toolbelt";
import { FilterQuery } from "mongoose";
import { Ctx, Next, Use } from "aom";
import { Controller, DelayRefStack, Err, ErrorFunction } from "aom";
import { Middleware, NextFunction, QueryParameters, Responses } from "aom";
import { REQUIRED_DATA_MISSING } from "common/constants";
import { Query, RouteRef, ThisRef, This, toJSONSchema } from "aom";
import { KoaContext } from "aom/lib/common/declares";
//
import { QueryFilter } from "./query-filter";
import { ErrorMessage, WrongDataResponse, SafeData } from "../api";
import { BaseModel } from "../schemas";
import { DataRoute } from "../routes/data-route";
// используем `ThisRef` так как хотим использовать данные последнего дочернего класса-контроллера
const SafeOrigin = ThisRef(<T extends typeof QueryBuilderConstructor>({ origin }: T) =>
  SafeData(origin)
);
const SafeOriginParameters = ThisRef(<T extends typeof QueryBuilderConstructor>(constructor: T) => [
  QueryParameters(...constructor.toJSON()),
]);
//
@Controller()
export class QueryBuilderConstructor<
  T extends typeof QueryFilter,
  R = C.Instance<T>,
  F = FilterQuery<R>
> extends QueryFilter {
  static origin;

  async queryBuild(query: R): Promise<F> {
    return { ...query } as unknown as F;
  }

  static generateJSONSchema() {
    return toJSONSchema(this.origin);
  }

  ctx: KoaContext;

  @Middleware()
  static async AttachContext(
    @Ctx() ctx: KoaContext,
    @This() self: QueryBuilderConstructor<typeof QueryFilter>,
    @Next() next: NextFunction
  ) {
    self.ctx = ctx;
    return next();
  }

  @Middleware()
  @Responses(WrongDataResponse.toJSON())
  static async NotEmpty(
    @Query(SafeOrigin) query: CustomData,
    @Err(ErrorMessage) err: ErrorFunction,
    @Next() next: NextFunction
  ) {
    if (!Object.entries(query).length) {
      return err(REQUIRED_DATA_MISSING, 400);
    }
    return next();
  }

  @Middleware()
  @DelayRefStack(SafeOriginParameters)
  @Responses(WrongDataResponse.toJSON())
  @Use(QueryBuilderConstructor.AttachContext)
  static async Search<T extends DataRoute<typeof BaseModel>>(
    @Query(SafeOrigin) query: CustomData,
    @This() self: QueryBuilderConstructor<typeof QueryFilter>,
    @This(RouteRef()) route: T,
    @Next() next: NextFunction
  ) {
    query = await self.queryBuild(query);
    Object.assign(route.where, { ...query });
    return next();
  }
}
//
export function QueryBuilder<T extends typeof QueryFilter>(origin: T) {
  return class extends QueryBuilderConstructor<T> {
    static origin = origin;
  };
}
```

Таким образом становится доступна возможность создавать сложные запросы, которые в том числе могут
обращаться к другим моделям данных, и обогатят значение `where` составным значением.

**Составление списка `{$in}`**

```ts
// `api/receipts/init.ts`

export class ReceiptsServicesFilterQuery extends QueryFilter {
  @IsEnum(ReceiptsServicesTypes, { each: true })
  @Expose()
  @TransformToArray()
  @IsOptional()
  @JSONSchema({ description: "Тип" })
  type: ReceiptsServicesTypes[];
}

@Controller()
export class ReceiptsServicesFilter extends QueryBuilder(ReceiptsServicesFilterQuery) {
  async queryBuild(query: ReceiptsServicesFilterQuery) {
    const where: FilterQuery<documentDataType> = {};
    if (_.size(query.type)) {
      where.type = { $in: [...query.type] };
    }
    return where;
  }
}
```

**Использование контекста с данными текущего пользователя**

```ts
// `api/invites/init.ts`

export class GroupsInvitesFilterQuery extends QueryFilter {
  @QueryBooleanInteger()
  @IsOptional()
  @JSONSchema({ description: "Заявки только для меня" })
  forMe?: boolean;

  @QueryBooleanInteger()
  @IsOptional()
  @JSONSchema({ description: "Заявки только от меня" })
  fromMe?: boolean;
}

@Controller()
export class GroupsInvitesFilter extends QueryBuilder(GroupsInvitesFilterQuery) {
  async queryBuild(query: GroupsInvitesFilterQuery) {
    const where: FilterQuery<schemas.GroupsInvites> = { $and: [] };
    const { $and } = where;
    const { ctx } = this;
    const { $StateMap } = ctx;
    // можно использовать данные из контекста `koa`
    const { userId } = $StateMap.get(Account) as Account;
    // заявки "для меня" - текущий пользователь - получатель заявки
    if (query.forMe) {
      $and.push({ invitedUserId: userId });
    }
    // заявки "от меня" - заявки где автор - текущий пользователь
    if (query.fromMe) {
      $and.push({ userId });
    }
    if ($and.length > 0) {
      Object.assign(where, { $and });
    }
    return where;
  }
}
```

**Сложный поиск в связанных моделях**

```ts
// `api/publications/init.ts`

export class PubsFilterQuery extends QueryFilter {
  @Expose()
  @IsMongoId()
  @IsOptional()
  @JSONSchema({ description: "ID пользователя" })
  userId: Types.ObjectId;

  @Expose()
  @IsMongoId()
  @IsOptional()
  @JSONSchema({ description: "ID кто подписан на указанного пользователя" })
  subscribersOf: Types.ObjectId;

  @Expose()
  @IsMongoId()
  @IsOptional()
  @JSONSchema({
    description: "ID тех, на кого подписан указанный пользователь",
  })
  subscriptionsOf: Types.ObjectId;

  @Expose()
  @IsString()
  @Sanitize()
  @IsOptional()
  @JSONSchema({ description: "Поиск по имени пользователя" })
  username: string;

  @Expose()
  @IsEnum(PublicationsAttachmentsReferences)
  @IsOptional()
  @JSONSchema({ description: "С приложениями указанного типа" })
  withAttachment: PublicationsAttachmentsReferences;
}

@Controller()
export class PubsFilter extends QueryBuilder(PubsFilterQuery) {
  async queryBuild(query: PubsFilterQuery) {
    const { ctx } = this;
    const { $StateMap } = ctx;
    const account: Account = $StateMap.get(Account);
    const { userId } = account;
    const where: FilterQuery<schemas.Publications> = { $and: [] };
    const { $and } = where;

    if (query.username) {
      const _id = [];
      // составление регулярной строки для поиска значений
      const nameParts = _.split(query.username.replace(/\?,\*,\./g, ""), " ").map(_.trim);

      const regExp = new RegExp(`${nameParts.join("|")}`, "i");
      const loginsUsersId = await models.UsersLogins.aggregateToSet(
        {
          type: LoginTypes.Plain,
          enabled: true,
          value: regExp,
        },
        "userId"
      );
      _id.push(...loginsUsersId);
      const usersId = await models.Users.aggregateToSet(
        {
          $or: [{ name: regExp }, { surname: regExp }],
        },
        "_id"
      );
      _id.push(...usersId, null);
      // добавим в список авторов тех, кто совпал по имени
      $and.push({ userId: { $in: _id } });
    }
    // найдем пользователей, на кого подписан указанный пользователь
    if (query.subscriptionsOf) {
      const _id = [];
      const userId = new Types.ObjectId(query.subscriptionsOf);
      const usersId = await models.UsersSubscribes.aggregateToSet({ userId }, "toUserId");
      _id.push(...usersId, null);
      $and.push({ userId: { $in: _id } });
    }
    // найдем пользователей, которые подписаны на указанного пользователя
    if (query.subscribersOf) {
      const _id = [];
      const toUserId = new Types.ObjectId(query.subscribersOf);
      const usersId = await models.UsersSubscribes.aggregateToSet({ toUserId }, "userId");
      _id.push(...usersId, null);

      $and.push({ userId: { $in: _id } });
    }

    if (query.withAttachment) {
      const referenceName = query.withAttachment;
      const publicationsId = await models.PublicationsAttachments.aggregateToSet(
        { referenceName },
        "publicationId"
      );
      // добавим в список идентификаторов те записи, которые имеют соответствующее вложение
      $and.push({ _id: { $in: publicationsId } });
    }

    if ($and.length > 0) {
      Object.assign(where, { $and });
    }
    return where;
  }
}
```

Созданные контроллеры подключаются через `middleware` к соответствущему контроллеру `DataRoute`.

```ts
import { ReceiptsServicesFilter } from "./init";

import { ReceiptsService } from "./service";

@Controller()
@Use(ReceiptsServices.Init)
@Bridge(`/service_${ReceiptsService}`, ReceiptsService)
export class ReceiptsServices extends CreateDataRoute(schemas.Receipts) {
  @Middleware()
  static async Init(@This() self: ReceiptsServices, @Next() next: NextFunction) {
    // подготовим данные в запросе
    self.where = { enabled: true };

    return next();
  }

  @Get()
  @Summary("Список кассовых сервисов")
  @Use(ReceiptsServicesFilter.Search)
  @UseNext(Renders.TotalData)
  static async Index(@This() self: ReceiptsServices, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }
}
```
