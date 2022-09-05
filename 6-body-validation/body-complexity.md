# Обработка составных структур

Составные структуры данных как правило содержат информацию о большом наборе контекстно связанных данных,
которые в общем случае нуждаются в особой логической обработке: проверка существования данных, промежуточное
удаление или обновление отсутствующих значений, специфические действия на наличие определенного значения.

В общем случае такая структура может выглядеть как перечисление нескольких типовых схем, связанных
определенным образом.

```ts

// POST /api/publication/{{id}}/set
{
  "publication": {
    "title": "short string"
    "content": "long string with extra data"
    "enabled": true,
  },
  "attachments": [
    {"referenceName": "Files", "referenceId": "62de67aa2fb90280163e6603"},
    {"referenceName": "Messages", "referenceId": "62e29b28de387de10709b755"},
    {"referenceName": "Videos", "referenceId": "62e5905b90ce7f91ce7f1825"},
  ]
}

```

Таким образом, что каждое значение корневой структуры относится к разным схемам данных, и необходимо обеспечить
их надежную последовательную проверку перед применением к базе данных.

Задача в общем случае сводится к созданию обособленных классов, которые объединяются в единую схему.

```ts
@ComponentSchema()
export class Publication {
  @IsString()
  @Expose()
  name: string;

  @IsString()
  @Expose()
  data: string;

  @IsBoolean()
  @Expose()
  @IsOptional()
  enabled: boolean;
}

@ComponentSchema()
export class Attachments {
  @IsEnum(AttachmentsReferences)
  @Expose()
  referenceName: AttachmentsReferences;

  @IsMongoId()
  @Expose()
  referenceId: Types.MongoId;
}

@ComponentSchema()
export class PublicationSetBody {
  @ValidateNested()
  @Type(() => Publication)
  @Expose()
  pubication: Publication;

  @ValidateNested({ each: true })
  @Type(() => Attachments)
  @Expose()
  @IsOptional()
  attachments: Attachments[];
}

@Controller()
export class PublicationSet extends BodyBuilder(PublicationSetDTO) {}
```

При этом возможно управление отдельно взятыми частями составной структуры, например, через эндпоинты вида
`POST /api/content/_{id}/attachments`, `PATCH /api/content/_{contentId}/attachments/_{attachmentId}`, и иной
степени детализации.

Таким образом, что хорошим решением такой ситуации становится возможность создания совместимых контроллеров,
способных вызываться друг из друга с передачей данных согласно контексту фукционала.

Существует общий случай решения такой задачи, который может иметь логическую вариативность в деталях.
Важной его особенностью является возможность комбинирования способов обращения к методам контроллеров,
и соответствующей реакцией на ошибки.

Главной логической последовательностью такого решения всегда будет: сначала полная проверка входящих данных,
а затем их применение к базе данных (удаление нерелевантных, добавление новых, обновление существующих).

## Передача данных между контроллерами

Существующая методология позволяет использовать совокупности контроллеров, которые обрабатывают некие
пограничные состояния данных, в качестве унифицированных элементов для логической обработки локального
фрагмента сложносоставных данных с поддержкой собственных аргументов или общего контекста.

При этом доступны 2 возможных сценария использования таких вызов: в качестве аргумента `next`-функции
или прямым обращением к методу.

Расмотренный ниже пример демонстрирует такую возможность: к получаемой схеме данных применяется вызов
типовой процедуры, используемой для единичного элемента, который может управляться собственными методами
контроллеров. Для создания новых значений используется непосредственный вызов метода требуемое количество
раз, обеспечивающего безошибочное выполнение на уже проверенных данных.

```ts
@Controller()
@Use(PublicationID.PathID)
@Bridge("/attachments", PublicationAttachments)
export class Publication {
  @Get()
  @Endpoint()
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

  /*
  пример сложно-составного обновления значения, когда из единой структуры применяется набор значений
  для разных сущностей, и каждую из них последовательно проверяет уже существующий код на `aom`, который
  применим для одного аналогичного значения из специализированного метода
  */
  @Post("/set")
  @Summary("Установка полного значения")
  @Use(PublicationSet.Body)
  @UseNext(Publication.Index)
  static async Set(
    @This(PublicationID) publication: PublicationID,
    @This(PublicationSet) { body }: PublicationSet,
    @This(PublicationAttachment) attachment: PublicationAttachment,
    @This(PublicationAttachments) attachments: PublicationAttachments,
    @Next() next: NextFunction
  ) {
    // проверим, чтобы все вложения были корректны
    await Promise.each(body.attachments, async (attachBody) => {
      attachment.body = attachBody;
      await next(PublicationAttachment.CheckBody);
    });
    // если мы оказались здесь, значит все вложения верны, и можно удалить имеющиеся
    // и добавить новые: новая проверка на существование не вызовет ошибки
    const publicationId = publication._id;
    attachments.where = { publicationId };
    await attachments.model.deleteMany(attachments.where);
    // для каждого вложения создадим новое в указанном контексте
    await Promise.map(body.attachments, async (attachBody) => {
      // результат нам не важен, подразумевается, что все произойдет без ошибок,
      // так как все данные уже проверены
      await PublicationAttachments.Add({ body: attachBody }, attachment, attachments, next);
    });
    // заменим собственное значение публикации
    publication.body = body.publication;
    return next(PublicationID.SaveBody, PublicationID.Define);
  }
}
```

Для передачи данных используется как присвоение локальному экземпляру класса, который будет использован
в качестве контекстного значения в серии типичных обращений к `endpoint`-у или `middleware`.

Данный способ управления данными является скорее конструктивной особенностью, чем рекомендацией к применению.
Логика таких конструкций объективно сложна и требует особого внимания при будущем рефакторинге, однако
в ряде случаев это позволит обеспечить совместимость сложных схем данных, которые могут использоваться согласно
особенностям бизнес-логики.
