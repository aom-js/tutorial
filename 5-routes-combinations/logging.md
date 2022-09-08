# Логирование данных

Методология `aom` позволяет контроллировать отдельные фрагменты маршрутов, позволяя
интегрировать механики логирования и сбора данных, которые используется в контексте
отдельно взятых методов.

Приведенный пример рассматривает общий случай создания логгера, который может быть
подключен к произвольным узлам маршрута и собирать обобщенную и фрагментарную информацию.

В числе собираемых данных может быть как содержимое пользовательского контекста (`body`,
`query-string`, `cookies`, `session`), так и значения контроллеров на маршруте, так и
расчеты времени выполнения относительно предыдущей операции замера времени и относительно
начала выполнения запроса.

Такие данные могут собираться в консолидированную базу данных типа `Elastic` или `Hadoop`,
и использоваться в дальнейшем для анализа, построения метрик и других исследований.

## Контроллер `Logger`

Логирование осуществляется на контроллере, который создает свой контекст в начале маршрута,
и повторное обращение к которому будет изменять его состояние и расширять состав данных.

```ts
import { Context, Request } from "koa";
import { nanoid } from "nanoid";
import { logger } from "config";
import apiLogger from "config/api-logger";
import { Controller, Ctx, Cursor, ICursor, IRoute, Middleware } from "aom";
import { Next, NextFunction, Route, StateMap, This } from "aom";

class LoggerAttachments {
  result;

  constructor(constructor: any, stateMap: WeakMap<any, any>) {
    this.result = () => stateMap.get(constructor);
  }

  toJSON() {
    return Reflect.apply(this.result, null, []);
  }
}

@Controller()
export class Logger {
  request: Request; // данные запроса

  idempotenceKey: string; // уникальный ключ запроса

  currentDate: Date; // дата записи

  startDate: Date; // время начала запроса

  finishDate: Date; // время завершения запроса

  route: IRoute; // выполняемый маршрут

  status: number; // итоговый статус

  lifetime: number; // время жизни запроса

  duration: number; // длительность выполнения запроса

  error: Error; // ошибка в запросе

  attachments = {}; // приложения

  constructor() {
    this.idempotenceKey = nanoid();
    this.startDate = new Date();
    this.lifetime = 0;
    this.duration = 0;
  }

  @Middleware()
  static async Init(
    @Route() route: IRoute,
    @This() log: Logger,
    @Ctx() ctx: Context,
    @Next() next: NextFunction
  ) {
    // в общем случае onerror всегда выполняется в конце, что позволяет мне зафиксировать время окончания запроса
    // и статус его завершения, и, в случае ошибки, саму ошибку
    ctx.onerror = () => {
      log.currentDate = new Date();
      log.finishDate = new Date();
      log.status = ctx.status;
      log.lifetime = +log.currentDate - +log.startDate; // время жизни запроса
      log.duration = +log.finishDate - +log.startDate; // длительность запроса

      if (log.status !== 200) {
        log.error = ctx.body as Error;
        apiLogger.error("request error", log);
      } else {
        apiLogger.info("request success", log);
      }
    };

    log.currentDate = new Date();

    log.request = ctx.request;

    log.route = route;

    Object.assign(ctx, { log });
    apiLogger.info("request initiated", log);
    return next();
  }

  @Middleware()
  static async Attach(
    @Cursor() cursor: ICursor,
    @StateMap() stateMap: WeakMap<any, any>,
    @This() log: Logger,
    @Next() next: NextFunction
  ) {
    const { origin } = cursor;
    // добавить в приложения к логированию указанный middleware/enpoint
    // извлекает origin, приложенный к курсору, и создает сущность, которая в момент составления лога извлекает
    // значение из stateMap
    // /*
    Object.assign(log.attachments, {
      [origin.constructor.name]: new LoggerAttachments(origin.constructor, stateMap),
    });
    // */
    return next();
  }

  @Middleware()
  static async Watch(@This() log: Logger, @Cursor() cursor: ICursor, @Next() next: NextFunction) {
    log.currentDate = new Date();
    log.lifetime = +log.currentDate - +log.startDate;
    const { origin } = cursor;
    const fragment = {
      prefix: cursor.prefix,
      constructor: origin.constructor.name,
      property: origin.property,
    };
    apiLogger.info("request fragment", { ...log, fragment });
    return next();
  }
}
```

Таким образом становится доступна следующая логика его использования

```ts
@Controller()
@Use(Logger.Init)
export class Root extends RootRoute {
  //...
}
// ...
@Controller()
@Use(Logger.Attach)
export class Account {
  //...
  @Middleware()
  @Use(Logger.Attach)
  static async Init() {
    // ...
  }

  //...
  @Middleware()
  @Use(Logger.Watch)
  static async Subscribe() {
    // ...
  }
}
```

Логика подключения логгера просходит следующим образом: за счет использования имени конструктора
на оригинал текущего курсора происходит создание контекстного накопителя данных, передающих его в
установленный модуль логирования (`winston`, `tslog` и тому подобные). Задача сводится к созданию
полезной управлющей структуры, в который будут сохранены значения начальные значения, определен
момент окончания всех действий и подсчет длительности операции за счет разницы действий.

Каждый вызов модуля логирования реагирует на свой уровень ошибки, который влияет на то, в какой файл
будет происходить запись.

```ts
// `config/api-logger.ts`

import fs from "fs-extra";
import winston, { Logger } from "winston";

const { LOGS_DIR } = process.env;

const errorLogs = `${LOGS_DIR}/api-error.log`;
const commonLogs = `${LOGS_DIR}/api-combined.log`;

// создадим потоки для записи данных
const errorLogsStream = fs.createWriteStream(errorLogs, { flags: "a+" });
const commonLogsStream = fs.createWriteStream(commonLogs, { flags: "a+" });

const logger: Logger = winston.createLogger({
  // silent: true,
  format: winston.format.json(),
  // defaultMeta: { service: 'user-service' },
  transports: [
    //
    // - Write all logs with level `error` and below to `error.log`
    // - Write all logs with level `info` and below to `combined.log`
    //
    new winston.transports.Stream({
      stream: errorLogsStream,
      level: "error",
    }),
    new winston.transports.Stream({
      stream: commonLogsStream,
    }),
  ],
});

export default logger;
```

Таким образом можно собирать статистику о том, какие данные были подключены к мониторингу, и за какое время
происходило какое-то наблюдаемое изменение состояния.

Конструкция `Logger.Attach` прикрепляет контекст текущего класса, и добавляет в результирующую структуру при
завершении запроса информацию о том, какие данные оказались в нем на момент завершения.

Конструкция `Logger.Watch` фиксирует значения в момент прохождения маршрута через эту точку и протоколировать
состав данных в наблюдаемой точке, если это требуется.

Генерируемые логи можно помещать в `Elastic` для дальнейшего анализа и построения графиков активности.
В качестве служебной информации в логах можно использовать произвольно взятые значения, которые могут требоваться
полезным контекстом: количество занятой процессом памяти, используемое дисковое пространство и так далее.