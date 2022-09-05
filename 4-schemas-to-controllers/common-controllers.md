# Общие контроллеры

Применение контроллеров над схемами как правило отображет некий частный случай работы с данными,
которое может быть естественным образом расширено в дочернем классе.

Таким образом становится доступна возможность типировать доступ к контроллерам на базе указанных схем данных.

```ts
// `common/models/controllers.ts`

/* eslint-disable prettier/prettier */
/* eslint-disable import/no-cycle */
import { BaseModel } from "common/schemas/base";
import * as schemas from "schemas";
import { CreateDataRoute, DataRoute } from "../routes/data-route";
import { CreateDocumentRoute, DocumentRoute } from "../routes/document-route";

// определим каждое свойство по отдельности, указав их фабриками классов
export class Routes<T extends typeof BaseModel> {
  data: () => typeof DataRoute<T>;

  document: (paramId?: string) => typeof DocumentRoute<T>;
}
// позволяет создать структуру данных над множеством схем, переданных по их ключам
export function buildModels<
  K extends keyof typeof schemas,
  S extends typeof schemas,
  T = { [N in K]: Routes<S[N]> }
>(schemas: S): T {
  const result = {};
  Object.keys(schemas).forEach((key) => {
    const schema = schemas[key];
    const routes = new Routes();
    routes.data = () => CreateDataRoute(schema) as any;
    routes.document = (paramId?: string) => CreateDocumentRoute(schema, paramId) as any;
    Object.assign(result, { [key]: routes });
  });
  return result as T;
}

// возвращает типичную структуру над всеми схемами
export default buildModels(schemas);
```

Таким образом становится доступна возможность создать указанный контроллер для схемы, получив
быстрый доступ к модели данных и свойствам ее параметризации данными из контекста.

```ts
// `api/categories/index.ts`
import { Post, Use, Summary, This, Bridge, UseNext } from "aom";
import { Controller, Get, Next, NextFunction } from "aom";

import { Category } from "./category";
import { AddCategoryRequest } from "./init";
import { controllers } from "common/models";

@Controller()
@Bridge(`/category_${Category}`, Category)
export class ItemsCategories extends controllers.ItemsCategories.data() {
  @Get()
  @Summary("Список категорий")
  @UseNext(ItemsCategories.TotalData)
  static async Index(@This() self: ItemsCategories, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }

  @Post()
  @Summary("Создать категорию")
  @Use(AddCategoryRequest.Body)
  @UseNext(Category.Index)
  static async Add(
    @This(AddCategoryRequest) { body }: AddCategoryRequest,
    @This(Category) category: Category,
    @Next() next: NextFunction
  ) {
    category.document = await category.model.create(body);
    return next();
  }
}
```

```ts
// `api/categories/category.ts`
import * as schemas from "schemas";
import { controllers } from "common/models";
import { Controller, Endpoint, Get, Next, NextFunction, Patch, Put } from "aom";
import { Responses, Summary, This, Use, UseNext } from "aom";
import { CreateDocumentRoute } from "common/routes";
import { AddCategoryRequest, PatchCategoryRequest } from "./init";

@Controller()
export class Category extends controllers.ItemsCategories.document("categoryId") {
  @Get()
  @Endpoint()
  @Summary("Информация о категории")
  @Responses(Category.toJSON())
  static async Index(@This() self: Category) {
    return self.document;
  }

  @Patch()
  @Summary("Изменить отдельные свойства")
  @Use(PatchCategoryRequest.Attach)
  @UseNext(Category.Index)
  static async Patch(@Next() next: NextFunction) {
    return next(this.SaveBody, this.Define);
  }

  @Put()
  @Summary("Перезаписать свойства")
  @Use(AddCategoryRequest.Attach)
  @UseNext(Category.Index)
  static async Put(@Next() next: NextFunction) {
    return next(this.SaveBody, this.Define);
  }
}
```

Такая нотация может быть применима для других смысловых областей работы над моделями: типовая статистика,
права доступа, экспорт-импорт, и другие случаи.
