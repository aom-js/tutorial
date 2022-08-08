# Ролевой доступ к данным

Подключение ролевых способов управления данными связано с понятием `маркеров`, описанных
в [`оригинальной документации`](https://aom.js.org/ru/docs/api/koa/middlewares/#marker).

Общая концепция решения такова: необходимо на списке маршрутов обозначить, что те или иные
ендпоинты ограничиваются неким механизмом контроля доступа, которому важно наличие для
пользователя тех записей, которая перечислены в списке этих индикаторов.

Создадим контроллер управления доступом.

```ts
// `api/access-control.ts`

export class AccessControl {
  model = models.Access;

  data: DocumentType<schemas.Access>[];

  accessId: Types.ObjectId[];

  accessPoints: AccessPoint[] = [];

  async init(userId): Promise<void> {
    this.accessId = await models.AccessUsers.aggregateToSet({ userId, enabled: true }, "accessId");
    this.data = await this.model.find({ _id: { $in: this.accessId }, enabled: true }).lean();
    // объединим все возможные доступы, которые предоставлены ему ролями
    this.data.forEach((access) => {
      this.accessPoints = _.union(this.accessPoints, access.points);
    });
  }

  checkAccess(point: AccessPoint): boolean {
    // функция проверки доступа: указанный маршрут должен
    return Boolean(_.find(this.accessPoints, _.pickBy(point, Boolean)));
  }

  // активируем контекст прав доступа пользователя
  @Middleware()
  static async Init(
    @This(Account) { user }: Account,
    @This() access: AccessControl,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    // ..
    await access.init(user._id);
    return next();
  }

  // мидлварь добавляющий точку контроля
  @Marker(AccessControl.setMark)
  @Middleware()
  @Responses(AccessDenyResponse.toJSON(ACCESS_DENIED))
  static async Required(
    @This() access: AccessControl,
    @Cursor() cursor: ICursor,
    @Route() route: IRoute,
    @Err(AccessDenyResponse) err: ErrorFunction,
    @Next() next: NextFunction
  ): Promise<ErrorResponse<ReturnType<NextFunction>>> {
    // ...
    const cursorIsEndpoint =
      cursor.constructor === route.constructor &&
      cursor.property === route.property &&
      cursor.handler === route.handler;

    const { prefix, method, path } = <AccessPoint>(cursorIsEndpoint ? route : cursor);

    return access.checkAccess({ prefix, method, path }) ? next() : err(ACCESS_DENIED, 403);
  }
  // имя маркера
  static markName = "AccessControl";
  // метод добавления маркера на маршрут
  static setMark(route: IRoute, cursor: ICursor): void {
    if (!route[this.markName]) {
      route[this.markName] = [];
    }

    const controlForEndpoint =
      cursor.origin.constructor === route.constructor && cursor.origin.property === route.property;

    const { prefix, method, path } = <AccessPoint>(controlForEndpoint ? route : cursor);

    route[this.markName].push({ prefix, method, path });
  }

  static toString(): string {
    return this.name;
  }
  // */
}
```

Созданный контроллер применяется к другим контроллерам и ендпоинтам, если требуется ограничить
доступ к ним на основании проверки полномочий из справочника.

```ts
@Controller()
@AddTag("Полномочия пользователя")
@Use(AccessControl.Required, UserPermissions.Init) // ограничим доступ ко всем endpoint-ам
export class UserPermissions {
  // ...
  permissions: UserPermissionsType;

  @Middleware()
  @UseTag(UserPermissions)
  static async Init(
    @This() data: UserPermissions,
    @This(User) { entity }: User,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    const { _id: userId } = entity;
    data.permissions = await models.UsersPermissions.findOne({ userId });
    return next();
  }

  @Get()
  @Summary("Полномочия пользователя")
  @Responses({
    status: 200,
    description: "Полномочия пользователя",
    schema: schemas.UsersPermissions,
  })
  static Index(@This() { permissions }: UserPermissions): UserPermissionsType {
    return permissions;
  }

  @Put()
  @Summary("Обновить полномочия пользователя")
  @RequestBody({ schema: UserPermissionDTO })
  @Responses({
    status: 200,
    description: "Полномочия пользователя",
    schema: schemas.UsersPermissions,
  })
  @Use(AccessControl.Required) // и отдельно ограничим доступ для обновления значений
  static async Update(
    @SafeBody(UserPermissionDTO)
    userRermissions: schemas.UsersPermissions,
    @This(User) { _id: userId }: User,
    @Next() next: NextFunction
  ): Promise<ReturnType<NextFunction>> {
    await models.UsersPermissions.updateOne(
      { userId },
      { $set: { ...userRermissions } },
      { upsert: true }
    );
    return next(this.Init, this.Index);
  }
}
```

Таким образом, что список роутеров (не в OpenAPI-документации, а в базовом ответе) будет содержать
маркерную информацию о тех точках, записи о которых необходимо иметь пользователю, чтобы обращаться к ней.

```json
[
  // ...
  {
    "method": "get",
    "path": "/api/users/user_:user_id([0-9a-fA-F]{24})/accessId",
    "AccessControl": [
      {
        "prefix": "/api"
      },
      {
        "prefix": "/api/users"
      },
      {
        "prefix": "/api/users/user_:user_id([0-9a-fA-F]{24})/accessId"
      }
    ],
    "operationId": "UserPermissions_Index"
  },
  {
    "method": "put",
    "path": "/api/users/user_:user_id([0-9a-fA-F]{24})/accessId",
    "AccessControl": [
      {
        "prefix": "/api"
      },
      {
        "prefix": "/api/users"
      },
      {
        "prefix": "/api/users/user_:user_id([0-9a-fA-F]{24})/accessId"
      },
      {
        "method": "put",
        "path": "/api/users/user_:user_id([0-9a-fA-F]{24})/accessId"
      }
    ],
    "operationId": "UserPermissions_Update"
  }
  // ...
]
```

Такая структура может быть визуализирована в тот или иной визуал (чаще всего древовидный), на котором
можно отметить доступ к значениям `{prefix}` и `{method,path}`, объединив их в именованную роль.

Модель данных, в которой хранятся такие значения, может быть представлена в виде

```ts
@JSONSchema({ description: "Роли доступа" })
@ComponentSchema()
export class Access extends EnabledModel {
  @prop()
  @Expose()
  @IsString()
  @JSONSchema({ description: "Название роли", example: "Модератор" })
  @Sanitize()
  name: string;

  @prop({ type: () => [AccessPoint], _id: false })
  @Expose()
  @IsOptional()
  @ValidateNested({ each: true })
  @Type(() => AccessPoint)
  points: AccessPoint[];
}

@ComponentSchema()
@JSONSchema({ description: "Точки доступа" })
export class AccessPoint {
  @prop()
  @Expose()
  @IsOptional()
  @IsString()
  @JSONSchema({
    description: "Метод endpoint-а",
    example: "get",
  })
  method?: string;

  @prop()
  @IsString()
  @IsOptional()
  @Expose()
  @JSONSchema({
    description: "Адрес endpoint-а",
    example: "/blog/post_:id",
  })
  path?: string;

  @prop()
  @IsString()
  @IsOptional()
  @Expose()
  @JSONSchema({
    description: "Префикс адреса",
    example: "/users",
  })
  prefix?: string;
}
```

То есть в общем случае роль хранит перечисление `{prefix}` и `{method,path}` значений, с которыми
сопоставляются характеристизующие свойства `middleware` и `endpoint`-ов.

Пользователь может иметь несколько ролей, которые могут расширять друг друга. Допустимо использование
нескольких разноплановых контроллеров доступа, если они подразумевают локальные роли в частном контексте:
управление пользователями или другими пользователями, товарами или другими данными.
