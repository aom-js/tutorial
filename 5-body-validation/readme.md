# Проверка входящих значений

Существует несколько типовых случаев обработки входящих данных в http-запросах:

- проверка существования документа
- создание нового документа
- полное или частичное редактирование имеющегося документа данных
- сложно-составная структура, использующая схемы нескольких моделей данных

В общем случае обработка входящих данных всегда сопоставляется с некоей схемой, использующей
типичные декораторы `class-validator` и `class-transformer`, которые и осуществляют безопасное
преобразование типов и проверяют полноту данных.

Чаще всего решается применением типового декоратора `@SafeBody`, который проверяет полноту и корректность
данных для входящего значения, и удаляет из него неиспользуемые значения.
Поведение `@SafeBody` есть частный случай над поведением `SafeData`. Применяемое правило обработки входящих
данных может быть изменено на локальное усмотрение. Используемая конфигурация по собственным наблюдениям
отвечает большинству требуемых кейсов.

```ts
// `common/api/safe-data.ts`
import { Body, Query } from "aom";
import { ClassConstructor, plainToInstance } from "class-transformer";
import { validate } from "class-validator";
import { WRONG_DATA_ERROR } from "common/constants";
import { WrongDataMessage } from "./error-messages";

export function SafeBody<T extends ClassConstructor<unknown>>(constructor: T): ParameterDecorator {
  return Body(SafeData(constructor));
}

export function SafeQuery<T extends ClassConstructor<unknown>>(constructor: T): ParameterDecorator {
  return Query(SafeData(constructor));
}

// fabric for safe data processing
export function SafeData<T extends ClassConstructor<unknown>>(constructor: T): Function {
  const handler = async (body) => {
    const safeAttrs = plainToInstance(
      constructor,
      { ...body },
      {
        exposeDefaultValues: true,
        strategy: "excludeAll",
        exposeUnsetFields: false,
        enableCircularCheck: true,
      }
    ) as T;

    const valueErrors = await validate(safeAttrs, {
      whitelist: true,
    });

    if (valueErrors.length) {
      // возвращает код 400, если форма не прошла проверку
      throw new WrongDataMessage(WRONG_DATA_ERROR, valueErrors);
    }
    return safeAttrs;
  };
  return handler;
}
```

При проектировании схем данных есть возможность корректно разделять свойства документов на те, которые ожидаются
для изменения, которые не должны быть показаны, которые только для чтения, или которых может не быть, используя
сочетания декораторов `@Expose` и `@Exclude`.

```ts
@ComponentSchema()
@JSONSchema({ description: "Произвольная модель данных" })
export class DataModel extends BaseModel {
  @prop({ index: true })
  @IsMongoId()
  @IsAllow() // чтобы при проверке входящих значений не было ошибки, примем любое из значений, так как его все-равно не будет в результате
  @JSONSchema({ description: "related document", readOnly: true })
  readonly attrId: Types.ObjectId;

  @prop({ required: true })
  @IsString()
  @Expose() // обязательное значение, ожидаемое из внешних источников
  @JSONSchema({ description: "name value" })
  public name: string;

  @prop()
  @IsDate()
  @IsOptional() // возможное значение
  @Expose() // ожидаемое из внешних источников
  @JSONSchema({ description: "birth date" })
  public birthDate?: Date;

  @prop()
  @IsMongoId()
  @Exclude() // данное значение будет исключено из схемы
  @JSONSchema({ description: "private key" })
  private keyId: Types.ObjectId;

  @prop({ type: () => ExtraProperties, _id: false })
  @ValidateNested() // вложенные значения
  @Type(() => ExtraProperties) // определенного типа
  @Expose() // ожидаемые извне
  @JSONSchema({ description: "extra properties" })
  subDocument: ExtraProperties;

  @prop({ ref: () => RelatedSchema, foreignField: "_id", localField: "attrId", justOne: true })
  @IsOptional() // возможный документ, который может содержать популяризированую информацию
  @ValidateNested()
  @Type(() => ExtraProperties)
  @JSONSchema({ description: "populated document", readOnly: true })
  readonly subDocument?: ExtraProperties;
}
```

Таким образом могут собираться пластичные схемы данных, которые удобно пропускать через сложносочетаемые
формы, таким образом, что логические фрагменты сложной обрабоки можно вызывать непосредственно цепочкой
методов контроллеров, и ожидать корректный результат в собственном значении контекстного объекта контроллера.

Результирующее применение такого подхода можно показать следующими примерами

### Стандартное добавление и редактирование данных

Пример разобран в файле [`body-standart`](./body-standart.md).
Используется в большинстве случаев обработки типовых или специальных форм.

### Сложно составные структуры

Пример разобран в файле [`body-complexity`](./body-complexity.md).
Может использоваться как стандарт храниения данных на весь контекст API, сочетает в себе лучшие из
механик, и в общем и целом совместим с типовой локальной обработкой форм.

## Передача данных между контроллерами

Существующая методология позволяет использовать совокупности контроллеров, которые обрабатывают некие
пограничные состояния данных, в качестве унифицированных элементов для с поддержкой собственных аргументов.

При этом доступна 2 возможных сценария использования таких вызов: в качестве значения `next`-функции
и прямым обращением к методу.

```ts
// найти более релевантный пример
```
