# Контроль доступа

Контроль доступа к данным чаще всего применяется в нескольких вариантах:

- авторизован или неавторизован пользователь
- имеет-ли пользователь специфические признаки и аттрибуты
- обладает ли пользователь ролью, имеющей специфическое разрешение к указанному маршруту
- иные ситуативные варианты: является ли участником группы или заблокирован по решению модератора

Подключение элементов контроля доступа происходит на уровне `middleware`, которые исходя из значений
в контексте http-запроса принимают решение: "пропустить" запрос дальше, или прервать его ошибкой.

## Проверка авторизации пользователя

Самый частый случай ограничения контроля - это проверка пользовательской сессии или токена.
Как правило являются `middleware` одного из верхних уровней, которые накладывают требования
для всех `endpoint`-ов в структуре маршрутов.

```ts
export class Auth {
  validToken: schemas.AuthTokens;

  @Middleware()
  @Responses(NotAuthResponse.toJSON(ERROR_AUTH_REQUIRED))
  static async Required(
    @Headers() headers,
    @This() self: Auth,
    @Error(NotAuthResponse) err: ErrorMessage,
    @Next() next: NextFunction
  ) {
    self.validToken = await models.AuthTokens.findByHeader(headers.authorization);
    if (self.validToken) {
      return next();
    }
    return err(ERROR_AUTH_REQUIRED);
  }
}
```

Подключение происходит на стадии уровне контроллера, начиная с которого следует использовать
авторизацию

```ts
@Use(Auth.Required, Account.Init)
@Bridge("/users", Users)
@Bridge("/customers", Customers)
@Bridge("/contents", Contents)
@Bridge("/www", Www)
@Bridge("/files", Files)
@Bridge("/payments", Payments)
@Bridge("/settings", Settings)
@Bridge("/items", Items)
@Bridge("/special", Special)
export class App {
  @Get()
  @Summary("Текущая сессия")
  @Responses(Account.toJSON())
  static Index(@This(Account) account: Account) {
    return account;
  }
}
```

Все маршруты подключенные к данному контроллеру будут проходить проверку авторизации пользователя
автоматически ввиду наличия `@Use(Auth.Required)`, и для всех них уже будет определен и создан контекст
значений, прошедших требуемые проверки.

## Наличие признаков у пользователя

Логика данного метода может отличаться от случая к случаю, однако чаще всего сводится к следующим случаям:

- наличие у пользователя определенного признака `isAdmin=boolean`, `permissions.Moderator=boolean` и т.п.
- наличие неких ассоциаций в других моделях данных: входит в определенную группу пользователей, обладает
  специальным предметом, не имеет активных банов и т.д.

В общем случае использование таких ограничений раскладывается на частные случаи, когда пользователь обращается
к контекстным данным определенной сущности, к которой можно иметь доступ только в особых случаях.

```ts
export class Account {
  login: schemas.UsersLogins;
  user: schemas.Users;

  @Middleware()
  static async Init(
    @This() self: Account,
    @This(Auth) { validToken }: Auth,
    @Next() next: NextFunction
  ) {
    self.user = await models.Users.findById(validToken.userId);
    self.login = await models.UsersLogins.findById(validToken.userLoginId);
    return next();
  }

  @Middleware()
  @Responses(AccessDenyResponse.toJSON(ERROR_ADMIN_REQUIRED))
  static async IsAdmin(
    @This() self: Account,
    @Err(AccessDenyResponse) err: ErrorFunction,
    @Next() next: NextFunction
  ) {
    if (user.isAdmin) {
      return next();
    }
    return err(ERROR_ADMIN_REQUIRED);
  }
}
```

Таким образом становится доступна возможность ограничивать определенные `endpoint`-ы для выполнения только тем
пользователям, которые обладают специфическими полномочиями.

```ts
@Use(Auth.Required, Account.Init, App.NotBanned)
@Bridge("/users", Users)
@Bridge("/customers", Customers)
@Bridge("/contents", Contents)
@Bridge("/www", Www)
@Bridge("/files", Files)
@Bridge("/payments", Payments)
@Bridge("/settings", Settings)
@Bridge("/items", Items)
@Bridge("/special", Special)
export class App {
  @Get()
  @Summary("Текущая сессия")
  @Responses(Account.toJSON())
  static Index(@This(Account) account: Account) {
    return account;
  }

  @Post("/reset")
  @Use(Account.IsAdmin)
  @Summary("Сбросить все сессии")
  @Responses(MessageResponse.toJSON())
  static Reset() {
    await models.AuthTokens.updateMany({ enabled: true }, { $set: { enabled: false } });
    return new MessageResponse("Все пользователи отключены");
  }

  @Middleware()
  @Responses(AccessDenyResponse.toJSON(ERROR_BAN))
  static async NotBanned(
    @This(Account) { user }: Account,
    @Err(AccessDenyResponse) err: ErrorFunction,
    @Next() next: NextFunction
  ) {
    const isBanned = await models.UsersBans.findOne({ enabled: true, userId: user._id });
    if (isBanned) {
      return err(ERROR_BAN);
    }
    return next();
  }
}
```

Ограничения на доступ к элементу данных определяется на уровне `middleware` контроллера, управляющего
данным контекстом.

```ts
// проверим, что пользователь является участником данной группы, чтобы получить информацию о ней

@Controller()
@Use(Group.IsAllowed)
export class Group extends controllers.Groups.document("groupId") {
  @Middleware()
  @Responses(AccessDenyResponse.toJSON(ERROR_NOT_IN_GROUP))
  static async IsAllowed(
    @This() self: UsersGroup,
    @This(Account) { user }: Account,
    @Err(AccessDenyResponse) err: ErrorFunction,
    @Next() next: NextFunction
  ) {
    //
    const userInGroup = await models.GroupsUsers.findOne({
      groupId: self._id,
      userId: user._id,
      leaveDate: null,
    });
    if (userInGroup) {
      return next();
    }
    return err(ERROR_NOT_IN_GROUP);
  }
  //.. Index, Patch, Delete, etc.
}
```

## Ролевой доступ к данным

Пример, рассмотренный в файле [`role-based`](./role-based.md) представляет собой общий случай
методологии внедрения ролевого доступа к отдельно взятым `endpoint`-ам, что позволяет также
разграничить доступ между большим числом пользователей в разных вариациях и пересечениях.

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
    @SafeBody(UserPermissionDTO) userRermissions: UserPermissionDTO,
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

Таким образом станет возможность создавать роли, которые получают доступ только на чтение значений
из контроллера `UserPermissions`, так и те, кто может записывать данные.

Элементы контроля данных могут быть задействованы на интерфейсах, чтобы показывать или блокировать тот или
иной элемент управления, логически отвечающий за взаимодействие с указанным эндпоинтом.
