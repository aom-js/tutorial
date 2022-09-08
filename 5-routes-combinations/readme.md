# Комбинация маршрутов

Возможности `aom` позволяют создавать сложные комбинаторные элементы, которые позволяют
контроллерам использовать свой собственный локальный контекст на любом участке маршрута.

Рассмотрим самые популярные случаи применения: использование аргумента из адресной строки.
Наиболее частые варианты обращения к этим данным выглядит как `/data/{dataId}/subdata/{subId}`,
либо как `/data/subdata?dataId=...&subId=...`.

Задача сводится к тому, чтобы при обращении к контроллеру, который уже инициирован на заранее известной
аргументации, в контексте вызова получать ожидаемое значение данных, в том числе расширенных.

```ts
@Controller()
@Bridge(`/${DataId}`, DataElem)
export class Data extends CreateDatRoute(schemas.Contents) {
  //
}

@Controller()
export class DataId extends CreateElemRoute(schemas.Contents, "dataId") {
  //
}

@Controller()
@Use(DataId.PathID)
@Bridge(`/subdata`, SubData)
export class DataElem {
  @Get()
  @Summary("Значение данных")
  static async Index(@This(DataId) { document }: DataId) {
    return document;
  }
}

@Controller()
@Bridge(`/${SubDataId}`, SubDataElem)
@Use(SubData.Init)
export class SubData extends CreateDatRoute(schemas.Comments) {
  // используем контекст значений
  @Middleware()
  static async Init(
    @This(DataId) { _id }: DataId,
    @This() self: SubData,
    @Next() next: NextFunction
  ) {
    self.where = { contentId: _id };
    return next();
  }

  @Get()
  @Summary("Значение контекстных данных")
  static async Index(@This(DataId) data: DataId, @This() self: SubData) {
    await self.getData();
    return self.data;
  }
}

@Controller()
export class SubDataId extends CreateElemRoute(schemas.Comments, "subId") {
  //
}

@Controller()
@Use(SubDataId.PathID)
export class SubDataElem {
  @Get()
  @Summary("Значение оригинальных и вложенных данных")
  static async Index(
    @This(SubDataId) { document: subDocument }: SubDataId,
    @This(DataId) { document: dataDocument }: DataId
  ) {
    return { subDocument, dataDocument };
  }
}
```

При корректной декомпозии файлов такой подход будет минимизировать количество циклических зависимостей,
и обеспечит гибкое использование элементов данных в контексте на вложенных конструкциях.

Подобная логика характерна и для использования аргументации `queryString`, использование которой обеспечит
возможность расширения поисковой аргументации из составных компонент.

```ts
@Controller()
@Use(SubDataId.QueryID)
export class SubDataElem {
  @Get()
  @Summary("Значение оригинальных и вложенных данных")
  static async Index(
    @This(SubDataId) { document: subDocument }: SubDataId,
    @This(DataId) { document: dataDocument }: DataId
  ) {
    return { subDocument, dataDocument };
  }
}
```

Возможно сочетание более сложных конструкций, каждая из которых использует собственную аргументацию,
разделяющую её контекст.

Такие контроллеры как правило состоят только из `middleware`, целью которых является переключения состояния
документа и реакции на определенные данные. Таким образом, что становится доступно использование общих методов
и единого контекста.

## Логирование данных

Пример, описанный к файле [`logging`](./logging.md) реализует возможность логировать произвольно
взятые участки маршрута, отслеживая в том числе время выполнения участка запроса и собираемые на
нем данные.

```ts
@Controller()
@Use(PublicationID.PathID)
export class Publication {
  @Get()
  @Endpoint()
  @Use(Logger.Attach)
  @Summary("Информация о публикации")
  @Responses(PublicationID.toJSON("Публикация"))
  static async Index(
    @This(PublicationID) publication: PublicationID,
    @This(PublicationAttachments) attachments: PublicationAttachments
  ) {
    attachments.where = { publicationId: publication._id };
    await attachments.getData();
    return { ...publication.document.toJSON(), attachments: attachments.data };
  }
}
```
