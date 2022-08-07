# Поиск данных

Поиск данных в моделях в общем смысле происходит за счет использования данных в строке
поиска `?query=string`, и решается использованием сочетаний контроллеров и схем данных.

Рассмотрим несколько типовых случаев, которые реализуют поставленные задачи на примере
контроллера `DataRoute`, которую требуется свести к безопасному наполнению значения
`self.where`, которое затем будет применено к типичному вызову `await self.getData()`.

## Поиск по схеме данных

Самый частый случай поиска данных - это использование значений схемы, которая используется
в качестве значений декоратора вида `@SafeQuery`.

При этом возможно, чтобы эта схема генерировала собственное `JSONSchema`-значение, которое
будет использоваться в `OpenAPI`-документации

```ts
// `common/classes/query-filter.ts`

import { toJSONSchema } from "aom";
import { OpenApiParameterObject } from "aom/lib/openapi/types";

// от этого класса могут наследоваться другие элементы, взаимодействующие со строкой поиска:
// сортировки, специальные фильтры, навигация со сдвигами и ограничениями
export class QueryFilter {
  // генерация схемы может отличаться в разных случаях
  static generateJSONSchema() {
    return toJSONSchema(this);
  }

  static toJSON(): OpenApiParameterObject[] {
    const jsonSchema = this.generateJSONSchema();

    const result = [];
    // используем структуру JSONShema
    Object.entries(jsonSchema.properties).forEach(([key, schema]) => {
      const required = jsonSchema.required?.indexOf(key) >= 0;
      const { description } = <any>schema;
      result.push({
        name: key,
        description,
        in: "query",
        schema,
        required,
      });
    });
    return result;
  }
}
```

Таким образом, что становится доступна возможность создавать классы, которые будут обогащать
контекст ендпоинтов использованием безопасных данных из их схемы.

```ts
// api/items/init.ts

export class ItemsSearch extends QueryFilter {
  @IsMongoId()
  @Expose()
  @IsOptional()
  @decorators.ItemsCategories.exists({ enabled: true })
  categoryId: Types.ObjectId;
}
```

Таким образом, что использование значений, прошедших безопасную проверку можно использовать
в качестве готовых значений для условий поиска.

```ts
// api/items/index.ts
import { ItemsSearch } from "./init";

@Controller()
@Bridge("/categories", ItemsCategories)
@Bridge(`/item_${Item}`, Item)
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

  @Get()
  @Summary("Список предметов")
  @Use(Items.Search)
  @UseNext(Items.TotalData)
  static async Index(@This() self: Items, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }
}
```

### Совмещение различных условий

Важной особенностью такого подхода является возможность совмещения различных фильтров, которые будут
ограничивать свою область поиска в типовой нотации. Для этого используются контроллеры с middleware.

Опишем некоторые типовые случаи использования типовой строки: поиск по `_id`, и навигационное смещение
`offset&limit`.

```ts
// `common/classes/query-filter.ts`

export class PagerFilter extends QueryFilter {
  @IsNumber()
  @Type(() => Number)
  @Expose()
  @IsOptional()
  @JSONSchema({
    description: "Смещение",
    default: 0,
  })
  offset = 0;

  @IsNumber()
  @Expose()
  @IsOptional()
  @Type(() => Number)
  @JSONSchema({
    description: "Ограничение количества",
  })
  limit;
}

export class IdFilter extends QueryFilter {
  @IsMongoId({ each: true })
  @Type(() => Types.ObjectId)
  @TransformToArray()
  @Expose()
  @IsOptional()
  @JSONSchema({
    description: "Идентификатор",
  })
  _id: Types.ObjectId[];

  @IsMongoId({ each: true })
  @Type(() => Types.ObjectId)
  @TransformToArray()
  @Expose()
  @IsOptional()
  @JSONSchema({
    description: "Пропускаемый идентификатор",
  })
  skip_id: Types.ObjectId[];
}
```

Создадим `middleware`, которые будут типичным образом использоваться в контексте класса `DataRoute`.

```ts
// `common/routes/query-string-data.ts`

import _ from "lodash";
import { Middleware, RouteRef, QueryParameters } from "aom";
import { This, Next, NextFunction, Use } from "aom";
import { BaseModel } from "common/schemas";
import { ErrorMessage, SafeQuery } from "common/api";
import { IdFilter, PagerFilter } from "common/classes/query-filter";
import { DataRoute } from "./data-route";

@Controller()
export class QueryStringData {
  @Middleware()
  @QueryParameters(...IdFilter.toJSON())
  static async SearchById<T extends DataRoute<typeof BaseModel>>(
    @SafeQuery(IdFilter) idFilter: IdFilter,
    @This(RouteRef()) route: T,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    const { _id = {} } = route.where;

    if (idFilter._id) {
      Object.assign(_id, { $in: idFilter._id });
    }
    if (idFilter.skip_id) {
      Object.assign(_id, { $nin: idFilter.skip_id });
    }
    if (Object.values(_id).length) {
      Object.assign(route.where, { _id });
    }
    return next();
  }

  @Middleware()
  @QueryParameters(...PagerFilter.toJSON())
  static async Pager<T extends DataRoute<typeof BaseModel>>(
    @This(RouteRef()) route: T,
    @SafeQuery(PagerFilter) pager: PagerFilter,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    route.pager = pager;
    return next();
  }
}
```

Таким образом становится доступна типичная логика навигации и поиска по данным

```ts
// `api/content/index.ts`

@AddTag("Материалы")
@Controller()
@Use(AccessAdmin.Required, Contents.Init)
@Bridge(`/content_${Content}`, Content)
@Bridge("/categories", ContentsCategories)
export class Contents extends CreateDataRoute(schemas.Contents) {
  @Middleware()
  @UseTag(Contents)
  @MergeNextTags()
  static Init(@This() contents: Contents, @Next() next: NextFunction): ReturnType<NextFunction> {
    contents.populates.add("category");
    return next();
  }

  @Get()
  @Summary("Список материалов")
  @Use(ContentsFilter.Search)
  @Use(QueryStringData.Pager, QueryStringData.SearchById)
  @UseNext(Contents.TotalData)
  static async Index(
    @This() contents: Contents,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    await contents.getData();
    return next();
  }
}
```

## Сложносоставные запросы

Пример разобран в файле [`query-builder`](./query-builder.md).
Позволяет использовать текущий контекст в момент генерации данных для поиска, выполнять
специфические проверки, создавать списки и составные ограничения.

```ts
import { PaymentsFilter } from "./init";

@Controller()
@Use(Payments.Init)
export class Payments extends CreateDataRoute(schemas.Payments) {
  @Middleware()
  static async Init(
    @This() self: Payments,

    @Next() next: NextFunction
  ) {
    // подготовим данные в запросе
    self.where = { enabled: true };
    self.populates.add({ path: "receipts" });
    self.sortOrder = { createdAt: -1 }; // самые свежие наверху
    return next();
  }

  @Get()
  @Summary("Список платежей")
  @Use(PaymentsFilter.Search)
  @UseNext(Payments.TotalData)
  static async Index(@This() self: Payments, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }
}
```
