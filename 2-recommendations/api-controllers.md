# Структура API контроллера

Типовой контроллер API чаще всего представляет собой логический элемент, работающий над определенной схемой данных.
Подробнее о том, как организовывать роутеры над схемами данных в
[соответствующем разделе](./../3-schemas-to-controllers/readme.md)

Типичной структурой узла `api` является совокупность файлов

```
-+ api/
--+ items/
---- brands/
---- categories/
---| data.ts
---| index.ts
---| init.ts
---| item.ts
```

При этом данные из `data.ts` и `init.ts` могут быть безопасно импортированы между разными узлами `api` и
использоваться в общих случаях. Таким образом достигается минимизация критических циклических зависимостей
между логическими узлами, и обеспечивается совместимость.

Подразумевается, что определенно взятый ендпоинт всегда возвращает полную безопасную информацию о своем
контексте, который можно использовать для составления сложных связей между документами. При этом все
`middleware`, которые были задействованы при формировании логической цепочки, формируют корректное ожидание
своей логики, и формируют информацию о себе в спецификации `OpenApi`.

Содержимое файлов будет удовлетворять следующему паттерну:

```ts
// `api/items/index.ts`

@Controller()
@Bridge(`/categories`, ItemsCategories)
@Bridge(`/brands`, ItemsBrands)
@Bridge(`/item_${Item}`, Item)
export class Items extends DataRoute(schemas.Items) {
  @Get()
  @Summary("Список данных")
  @Use(QueryString.Pager, ItemsQuery.Search, Items.Data)
  @Responses(Items.toJSON(["Список предметов"]))
  @UseNext(Renders.TotalData)
  static async Index(@This() self: Items, @Next() next: NextFunction) {
    return next();
  }

  @Middleware()
  static async Data(@This() self: Items, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }

  @Post()
  @Summary("Добавить предмет")
  @Use(ItemsData.StrictBody)
  @UseNext(Item.Index)
  static async Add(
    @This() self: Items,
    @This(ItemsData) { body }: ItemsData,
    @This(Item) item: Item,
    @Next() next: NextFunction
  ) {
    item.document = await self.model.create(body);
    return next();
  }
}
```

```ts
// `api/items/item.ts`

@Controller()
@Use(Item.Init)
export class Item extends DocumentRoute(schemas.Items) {
  @Middleware()
  @Responses(NotFoundResponse.toJSON())
  static async Init(@This() self: Item, @Next() next: NextFunction) {
    self.document = await self.model.findById(self._id);
    if (!self.document) {
      return new NotFoundResponse("предмет не найден");
    }
    return next();
  }

  @Get()
  @Endpoint()
  @Summary("Информация о предмете")
  @Responses(Item.toJSON("Информация о предмете"))
  static async Index(@This() self: Item): Item.documentType {
    return self.document;
  }

  @Get("/data")
  @Endpoint()
  @Summary("Данные о предмете")
  @Responses(ItemData.toJSON())
  static async Data(@This() self: Item): Item.documentType {
    const { categoryId, brandId } = self.document;
    const result = new ItemData();
    result.item = self.document;
    result.brand = await models.ItemsBrands.findById(brandId);
    result.category = await models.ItemsBrands.findById(categoryId);
    return result;
  }

  @Patch()
  @Summary("Изменить предмет")
  @Use(ItemsData.PartialBody, ItemsData.CheckPhotos)
  @UseNext(Item.SaveBody)
  static async Edit(
    @This(ItemsData) { body }: ItemsData,
    @This() self: Item,
    @Next() next: NextFunction
  ) {
    self.body = body;
    return next();
  }

  @Delete()
  @Summary("Удалить предмет")
  @UseNext(Item.SaveBody)
  static async Delete(@This() self: Item, @Next() next: NextFunction) {
    self.body = { deletedAt: new Date() };
    return next();
  }

  @Post("/restore")
  @Summary("Восстановить предмет")
  @UseNext(Item.SaveBody)
  static async Restore(@This() self: Item, @Next() next: NextFunction) {
    self.body = { deletedAt: null };
    return next();
  }

  @Delete("/destroy")
  @Summary("Уничтожить предмет")
  @Responses(MessageResponse.toJSON("Предмет уничтожен"))
  static async Destroy(@This() self: Item) {
    await self.model.deleteOne({ _id: self._id });
    return new MessageResponse("Предмет уничтожен");
  }

  @Endpoint()
  @UseNext(Item.Index)
  static async SaveBody(@This() self: Item, @Next() next: NextFunction) {
    await self.model.updateOne({ _id: self._id }, { $set: self.body });
    return next(this.Init);
  }
}
```

Для сложных вычислений рекомендуется использовать статические или собственные методы схем данных.
Если подразумевается сложно составной ответ, то корректно описать его схему в `init`-данных.

```ts
// `api/items/init.ts`

export class ItemsQuery extends CreateQueryFilter(schemas.Items, ["categoryId", "brandId"]) {}

@ComponentSchema()
export class ItemData extends CommonResponse {
  @JSONSchema({ description: "Предмет" })
  item: schemas.Items;

  @JSONSchema({ description: "Бренд" })
  brand: schemas.ItemsBrands;

  @JSONSchema({ description: "Категория" })
  category: schemas.ItemsCategories;
}
```

Содержимое `data`-файлов может быть специфическим для каждого логического случая, и общем случае
представляет собой набор `middleware` для проверки или генерации сложно-составных данных.

```ts
// `api/items/data.ts`
@Controller()
export class ItemsData extends BodyRoute(schemas.Items) {
  @Middleware()
  @Responses(WrongDataResponse.toJSON())
  static async StrictBody(
    @SafeBody(schemas.Items) body: schemas.Items,
    @This() self: ItemsData,
    @Next() next: NextFunction
  ) {
    self.body = body;
    return next();
  }

  @Middleware()
  @Responses(WrongDataResponse.toJSON())
  static async PartialBody(
    @SafeBody(PartialItemDTO) body: PartialItemDTO,
    @This() self: ItemsData,
    @Next() next: NextFunction
  ) {
    self.body = body;
    return next();
  }

  @Middleware()
  @Responses(WrongDataResponse.toJSON())
  static async CheckPhotos(
    @SafeBody(PartialItemDTO) body: PartialItemDTO,
    @This() self: ItemsData,
    @Next() next: NextFunction
  ) {
    const isValidPhotos = await models.Files.checkAllExists(body.filesId);
    if (!isValidPhotos) {
      return new WrongDataResponse("Ошибка состава данных");
    }
    return next();
  }
}
```
