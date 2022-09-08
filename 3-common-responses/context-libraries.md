# Контекстные библиотеки

Регулярным случаем создание API является наличие специальных служебных способов визуализации
данных, генерируемых сервером, в том числе ограничение доступа по частоте запроса. В общем случае
такие ответы могут быть генерируемы типовыми серверными библиотеками, чаще всего расширения к
`koa`.

В общем случае задача сводится к тому, чтобы подключить оригинальный контекст в процедуру библиотеки
и вернуть ответ, который бы корректно был обработан в цепочке `Use` на пути всего маршрута.

## Swagger

Для того, чтобы при создании сервера генерировать автоматический endpoint, позволяющий тестировать API,
создаемое в моменте времени на базе наполняемой документации, хорошо подходит подключение типовой
библиотеки `koa2-swagger-ui`.

Для того, чтобы сделать endpoint, который будет генерировать документацию, которая будет доступна по
унифицированной ссылке требуется в контроллер добавить соответствующий метод

```ts
import { koaSwagger } from "koa2-swagger-ui";

@Controller()
export class Swagger {
  @Get("/swagger.html")
  @Summary("OpenAPI Swagger UI")
  @Responses({
    status: 200,
    description: "Интерфейс для работы с документацией",
    schema: { type: "string" },
  })
  static Swagger(@This() { docs }: RootRoute, { ctx, next }): string {
    const swagger = koaSwagger({
      routePrefix: false,
      swaggerOptions: {
        spec: docs.toJSON(),
      },
    });
    return swagger(ctx, next) && ctx.body;
  }
}
```

Поскольку результатом функции `swagger` внутри эндпоинта будет изменение значения `ctx.body` оригинального контекста
и вызов оригинальной `next`-функции, то возвращемым значением будет именно это значение, которое будет интепретировано
как `html` со всеми готовыми данными.

## Ограничение количества запросов (защита от брутфорса)

Задача ограничения количества вызовов решается использованием библиотеки `koa2-ratelimit`. Для него следует
использовать конфигурационную конструкцию, которая возвращает результат выполнения настроенной `middleware`
на участке маршрута, к которому подключено ограничение доступа.

```ts
class RateLimitResponse extends ErrorMessage {
  static status = 429;

  static description = "Превышено количество запросов";
}

type RateLimitedRoute = Constructor & { rateLimiter: RateLimiter };

const storeClient = new Stores.Memory();

RateLimit.defaultOptions({
  store: storeClient,
});

interface IRateLimiter {
  safeRequests: number;
  maxRequests: number;
  inSeconds: number;
  waitSeconds: number;
}
@Controller()
export class RateLimiter implements IRateLimiter {
  safeRequests = 10;

  maxRequests = 20;

  inSeconds = 1;

  waitSeconds = 0;

  static prefixMap: Map<string, any> = new Map();

  constructor(params: IRateLimiter) {
    Object.assign(this, params);
    return this;
  }

  getMapMiddleware(prefix, originConstructor: RateLimitedRoute) {
    //
    if (!RateLimiter.prefixMap.has(prefix)) {
      const { rateLimiter = this } = originConstructor;
      const limiterMiddleware = RateLimit.middleware({
        interval: { sec: rateLimiter.inSeconds }, //
        delayAfter: rateLimiter.safeRequests, // begin slowing down responses after the first request
        timeWait: rateLimiter.waitSeconds * 1000, //
        max: rateLimiter.maxRequests, // start blocking after N requests
        prefixKey: prefix, // to allow the bdd to Differentiate the endpoint
        keyGenerator: (ctx) => {
          return prefix + (_.get(ctx, "session.uid") || ctx.get("x-real-ip") || ctx.ip);
        },
        handler: (...args) => {
          throw new RateLimitResponse(RateLimitResponse.description);
        },
      });
      RateLimiter.prefixMap.set(prefix, limiterMiddleware);
    }
    return RateLimiter.prefixMap.get(prefix);
  }

  @Middleware()
  @Responses(RateLimitResponse.toJSON())
  static async Attach(
    @Ctx() ctx: Context,
    @Next() next: NextFunction,
    @This() rateLimiter: RateLimiter,
    @Cursor() cursor: ICursor
  ) {
    const { prefix } = cursor;
    const middleware = rateLimiter.getMapMiddleware(
      prefix,
      <RateLimitedRoute>cursor.origin.constructor
    );
    const _next = await next();
    return middleware(ctx, _next);
  }
}
```

Таким образом, что становится доступна возможность ограничивать как отдельный `endpoint`, так и
обобщенную `middleware` по количеству запросов в указанное время.

Подключаемый контроллер может иметь статичное свойство `static rateLimiter: RateLimiter`, настроенное на
базовые ограничители `maxRequests` и `inSeconds`: количество запросов за указанное количество секунд.
Может быть донастроено на другие аргументы конструктора `RateLimit.middleware`.

Чаще всего подключается к сложной процедурной функции, которую не стоит дергать слишком часто

```ts
@AddTag("Авторизация")
@Controller()
export class LoginRoute extends AuthBase {
  // максимум 30 запросов в минуту, c паузой после 10-го в 2 секунды
  static rateLimiter = new RateLimiter({
    safeRequests: 10,
    maxRequests: 30,
    inSeconds: 60,
    waitSeconds: 2,
  });

  @Post("/request")
  @Summary("Запросить авторизацию по телефону")
  @RequestBody({ schema: RequestCodeForm })
  @Responses(ConflictResponse.toJSON(), RequestCodeResponse.toJSON("Код отправлен"))
  @Use(RateLimiter.Attach, LoginRoute.CheckLogin)
  static async RequestCode(
    @This() { login, permissions }: LoginRoute,
    @Err(ErrorMessage) err: ErrorFunction
  ): Promise<MessageResponse> {
    // ...
    return new MessageResponse(`Код отправлен`);
  }

  @Post("/confirm")
  @Summary("Подтвердить авторизацию по телефону")
  @RequestBody({ schema: ConfirmCodeForm })
  @Use(RateLimiter.Attach, LoginRoute.CheckLogin)
  @UseNext(LoginRoute.Token)
  @Responses(ConflictResponse.toJSON(), WrongDataResponse.toJSON(AUTH_LOGIN_WRONG_CONFIRM_CODE))
  static async ConfirmCode(
    @This() auth: LoginRoute,
    @SafeBody(ConfirmCodeForm) confirmBody: ConfirmCodeForm,
    @Err(ErrorMessage) err: ErrorFunction,
    @Next() next: NextFunction
  ): Promise<ErrorResponse<ReturnType<NextFunction>>> {
    // ...
    return next();
  }
}
```

Таким образом могут быть ограничены доступы к произвольно взятым сегментам данным, которым характерна значительная
нагрузка, но не характерно частое обращение клиентов, внедряя элементы защиты от перебора или атаки.
