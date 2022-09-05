# Расширение результирующих структур данных

Одной из возможностью, которой обладает объединение элементов контроллеров `aom`, является
допустимость типичной обработки произвольных данных для получения нового типичного ответа,
который можно использовать в обертке с другой типовой структурой, так и для радикального
преобразования типа `server-side-rendering`, или преобразование в xml или какое-то другое
типичное применение данных в известном контексте.

Чисто технически такая механика может обеспечить определенную степень технологической совместимости
при переносе на `aom` уже существующего функционала, использующего большое количество наработанных
инструментов, например, с частичным сохранением функционала на шаблонах, используемых для рендеринга.

В общем смысле механика сводится к применению `@UseNext()` конструкций, которые обеспечивают
совместимость со схемой управляющего контроллера. Данный метод активно использует функцию `RouteRef`.

Например, метод, который на основании контроллера, созданного на базе контроллера `DataRoute` может быть
обернут в конструкцию, которая дополняет информацию о составе данных общим количеством данных, которые
извлекаются по тому же запросу, по которым были получены значения `data`.

```ts
import { Controller, Endpoint } from "aom";
import { Responses, RouteRef, This } from "aom";
import { TotalDataResponse } from "common/api";
import { BaseModel } from "common/schemas";
import { DataRoute } from "./data-route";

const combineResponseSchema = (schema, attr = "data") =>
  RouteRef(<T extends typeof DataRoute>(route: T) => route.combineSchemas(schema, attr));

@Controller()
export class Renders {
  // ...
  @Endpoint()
  @Responses({
    status: 200,
    description: "Структура со сводной информацией",
    schema: combineResponseSchema(TotalDataResponse),
  })
  static async TotalData<T extends DataRoute<typeof BaseModel>>(
    @This(RouteRef()) route: T
  ): Promise<TotalDataResponse> {
    const { where, data, model } = route;
    const result = new TotalDataResponse();
    result.data = data;
    result.total = await model.countDocuments(where);
    return result;
  }
}
```

Использование такого типа преобразования данных происходит следующим образом:

```ts
@Controller()
export class Items extends CreateDataRoute(schemas.Items) {
  @Get()
  @Summary("Состав данных")
  @UseNext(Renders.TotalData)
  static async Index(@This() self: Items, @Next() next: NextFunction) {
    await self.getData();
    return next();
  }
}
```

Использование `RouteRef` позволяет обратиться к контроллеру, из которого данные были переброшены в `Renders`,
и использовать в контексте подключенного контроллера оригинальные данные, которые были подготовлены и сохранены.

В качестве результирующего преобразования может быть применение шаблона, который возвращает строку, или применение
другой специфической операции, которая типовым образом дополняет контекст типовых контроллеров.
