# Создание API на `aom`

Данный файл рассматривает пример создания `api`-сервиса на базе актуальной версии `aom`.

На настоящий момент времени это использование совокупности технологий: `typescript`, `koa@2`,
`MongoDB`, `Typegoose`. `aom` выступает декоративной оберткой над `koa-router`, и оперирует
файлами, которые принимают, фильтруют и единотипно оперируют http-запросами над данными.

То есть в общем случае контроллер `aom` представляет собой типовой логический модуль обслуживания
определенного типового контекста над справочниками базы данных: извлечение и изменение данных.

Готовые примеры кода доступны отдельным репозиторием, и могут быть использованы для решения
типичных задач и обеспечения миграции существующих решений в нотацию `aom`.

## Концепция

Существующая логика `aom` позволяет создавать монорепозитарный код, который может быть запущен как
в монолитной версии (для разработки), так и в микросервисной архитектуре, позволяющую распределить
нагрузку по ядрам, а также при необходимости запускать на разных инстансах (горизонтальное масштабирование).

В общем случае задача сводится к созданию набора контроллеров, логической задачей которых является вызов некоего
статического метода, который уже безопасно оперирует хранимым в нем контекстом.

Этот контекст может быть обогащен как данными `ctx`-значения из `koa`, так и внутри собственных вызовов, позволяя
повторно использовать сложные логические действия для получения ожидаемого значения.

Контроллеры чаще всего представляют собой обертки над моделями данных, и выполняют конечные обращения к методам
используемой `ORM`.

Таким образом естественным образом обеспечивается покрытие различных бизнес-кейсов, суть которых сводится к тому,
чтобы улучшать и развивать методы API, обеспечивая высокую степень совместимость с клиентами. В общем случае это
решение задачи генерации `OpenApi`-документации и типирования используемых структур данных. Это обеспечивает 
возможность комфортно обращаться к этим данным удобным способом, например, используя `RTK Query`.

## Организация файлов

Создание API лучше всего типируется конфигурацией его запуска. В общем случае - это порт, на котором
стартует сервис, и знание о неких ссылках, которые могут быть использованы в контексте (например, при генерации
уведомлений, в которых следует указать адрес, по которому следует перейти пользователю).

Лучше всего для этого подходит `.env`

```.env
NODE_ENV=development

# mongo db connections
DB_HOST=localhost
DB_NAME=scarych_ru

AUTH_SECRET_KEY=test

REDIS_PASSWORD=redis123

# BASIC_AUTH=test:test

## API settings
ADMIN_API_HOST=http://localhost:7100
ADMIN_API_PORT=7100

USERS_WEB_URL=http://localhost:3000
USERS_API_HOST=http://localhost:7300
USERS_API_PORT=7300

AUTH_API_HOST=http://localhost:7500
AUTH_API_PORT=7500

HOOKS_WEB_URL=https://floppy-turkeys-judge-94-25-168-238.loca.lt
HOOKS_API_HOST=http://localhost:7750
HOOKS_API_PORT=7750

FILES_WEB_URL=http://localhost:7550
FILES_API_HOST=http://localhost:7550
FILES_API_PORT=7550

# directories
FILES_DIR=./files
TMP_DIR=/tmp/files
LOGS_DIR=./logs

# kafka server
KAFKA_SERVER=localhost:9092

# kafka distributor instance key
KAFKA_INSTANCE_KEY=develop

# ELK stack security
KIBANA_SYSTEM_PASSWORD=oD*************Z6RD
KIBANA_ENCRYPTION_KEY=9abU***************tw9z
ELASTIC_PASSWORD=ubC************ZMt7

# server IP
SERVER_IP=127.0.0.1
```

Таким образом данный конфигурационный файл может быть совместим с `docker-compose.yml`, и поднимать контейнеры
с указанными параметрами.

```yaml
version: "3.7"
services:
  zookeeper:
    image: zookeeper
    restart: "unless-stopped"
    tmpfs: "/datalog"
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    restart: "always"
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - "zookeeper"
    command:
      - "/bin/sh"
      - "-c"
      - 'echo "wait some time..." && /bin/sleep 10 && start-kafka.sh' # run kafka after sleep time for connect to zookeeper

  redis:
    image: redis
    restart: "unless-stopped"
    environment:
      - ALLOW_EMPTY_PASSWORD=no
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - REDIS_DISABLE_COMMANDS=FLUSHDB,FLUSHALL
    ports:
      - 6379:6379
    volumes:
      - redis-volume:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf --requirepass ${REDIS_PASSWORD}
  prerender:
    image: tvanro/prerender-alpine
    ports:
      - "8887:3000"
    init: true
    environment:
      MEMORY_CACHE: 1
      CACHE_MAXSIZE: 1000
      CACHE_TTL: 3600 # 1 hour cache ttl
  cors-anywhere:
    image: testcab/cors-anywhere
    ports:
      - "8888:8080"
      # environment:
      # CORSANYWHERE_WHITELIST: http://localhost:3000

volumes:
  redis-volume:
    driver: local
```

Подразумевается, что API знают о существовании друг друга по указанным значениям, и могут использовать
данные внутри контекста бизнес-логики.

API создается отдельным каталогом и могут использовать общие данные для безопасного обмена информацией
между клиентом и сервером, так и между модулями.

```
-+ src/
--+ api/
---- admin/
---+ auth/
----+ routes/
-----| index.ts
-----| init.ts
----| docs.ts
----| index.ts
----| init.ts
----| root.ts
---- files/
---- users/
---- hooks/
---| index.ts
---| server.ts
--+ common/
---- api/
---- functions/
---- classes/
--- schemas/
--- kafka/
-+ scripts/
--| development.ts
--| addsuperuser.ts
-| .env
-| package.json
-| ecosystem.config.js
-| tsconfig.json
```

Запустить сервисы можно как все вместе, и так в индивидуальных процессах.

```ts
// `./src/api/index.ts`
export "./auth"
export "./admin"
export "./users"
export "./files"
export "./hooks"
```

```ts
// `ecosystem.config.js`
// PM2 startup script
const { NODE_ENV } = process.env;

const defaults = {
  merge_logs: NODE_ENV !== "development",
  log_date_format: "MM/DD/YYYY HH:mm:ss",
  env: {
    NODE_ENV,
    NODE_PATH: "./build/src",
  },
};

module.exports = {
  apps: [
    {
      name: "auth-api",
      out_file: "/tmp/auth-api.log",
      script: "build/src/api/auth/index.js",
      ...defaults,
    },
    {
      name: "files-api",
      out_file: "/tmp/files-api.log",
      script: "build/src/api/files/index.js",
      ...defaults,
    },
  ],
};
```

Все API-сервисы создают экземпляр `aom`-контейнера, который передается в http-сервер на базе `koa`.

```ts
// `./src/api/server.ts`
import koa from "koa";
import body from "koa-body";
import koaRouter from "koa-router";
import websocket from "koa-easy-ws";
import { $ } from "aom";
import cors from "@koa/cors";
import { NotFoundResponse } from "common/api";

export interface IApiSettings {
  port: number;
  url?: string;
  origin?: string;
}

export default class Server {
  constructor($aom: $, settings: IApiSettings) {
    const server = new koa();
    server.use(body({ multipart: true, formLimit: "50mb" }));
    server.use(websocket());
    server.use(cors({ credentials: true }));

    const router = new koaRouter();

    $aom.eachRoute(({ method, path, middlewares }) => {
      router[method](path, ...middlewares);
    });

    server.use(router.routes()).use(router.allowedMethods());

    // standart 404 response if path wasn't found
    server.use((ctx) => {
      ctx.body = new NotFoundResponse();
      ctx.status = 404;
    });
    server.listen(settings.port);
    console.info("server started", { settings });
  }
}
```

Каждый возвращаемый модуль API представляет собой экземпляр класса `Server`, создание которого означает
корректность подключенных маршрутов `$aom` на указанном порту и установленным окружением.

Файл `server.ts` расширяется если требуется обогатить `koa`-контекст единым образом для всех api: сессии,
параметры безопасности, логгеры и так далее.

Типовой `aom` модуль создает экземпляр `$aom` с подключением маршрутов и документации

```ts
// `src/api/auth/index.ts`
import Server from "api/server";
import { settings } from "./init";
import { $root } from "./root";

export const server = new Server($root, settings);
```

```ts
// `src/api/auth/init.ts`
const { AUTH_API_HOST, AUTH_API_PORT } = process.env;

if (!(AUTH_API_HOST && AUTH_API_PORT)) {
  throw new Error(".env variables required: AUTH_API_PORT, AUTH_API_HOST");
}

export const url = AUTH_API_HOST;

export const settings = {
  url,
  port: +AUTH_API_PORT,
  env: { AUTH_API_HOST, AUTH_API_PORT },
};
```

```ts
// `src/api/auth/docs.ts`
import { OpenApi } from "aom";
import { url } from "./init";

const Doc = new OpenApi({
  info: {
    title: "Auth api documentation",
    description: "Документация для API авторизации",
    contact: {
      name: "Kholstinnikov Grigory",
      email: "mail@scarych.ru",
    },
    version: "1.0.0",
  },
  servers: [
    {
      url,
      description: "current host",
    },
  ],
  openapi: "3.0.1",
});

export default Doc;
```

```ts
// `src/api/auth/root.ts`
import { Responses, Summary, This, Use } from "aom";
import { $, Bridge, Controller, Post } from "aom";
import models from "common/models";
import { Auth, RootRoute } from "common/routes";
import { AccessDenyResponse, MessageResponse } from "common/api";
import Docs from "./docs";
import { url } from "./init";
import { LoginRoute } from "./routes";

@Bridge("/", LoginRoute)
@Controller()
class Root extends RootRoute {
  routes = $root.routes;

  docs = Docs;

  url = url;

  @Post("/logout")
  @Use(Auth.Required)
  @Summary("Завершить сессию")
  @Responses(MessageResponse.toJSON())
  static async Logout(@This(Auth) auth: Auth) {
    //
    const { _id: tokenId } = auth.token;
    await models.UsersLoginsTokens.updateOne({ _id: tokenId }, { $set: { enabled: false } });

    return new MessageResponse("Сессия завершена");
  }
}

export const $root = new $(Root).docs(Docs);
```

Таким образом создается корневой контроллер, к которому могут быть подключены дополнительные маршруты через
декоратор `@Bridge`.

Активно используется наследование от общих контроллеров, которые позволяют использовать общие действия в
типовом контексте. Например, создавать типовые ответы с документацией и списком поддерживаемых методов.

```ts
// `common/routes/root.ts`
import { Responses, This, IRoute, Use, ComponentSchema, Endpoint } from "aom";
import { Controller, Get, OpenApi } from "aom";
import { Summary } from "aom";
import { koaSwagger } from "koa2-swagger-ui";
import { getDisplayName } from "aom/lib/special/display-name";
import { IsString } from "class-validator";
import { JSONSchema } from "class-validator-jsonschema";
import { BasicAuth } from "./basic-auth";
import { Logger } from "./logger";

if (!process.env.SERVER_IP) {
  throw new Error("process.env.SERVER_IP required");
}

@ComponentSchema()
export class RouteElement {
  @IsString()
  @JSONSchema({ description: "HTTP метод" })
  method: string;

  @IsString()
  @JSONSchema({ description: "Путь" })
  path: string;

  @IsString()
  @JSONSchema({ description: "ID действия" })
  operationId: string;
}

@Controller()
@Use(Logger.Init)
export class RootRoute {
  static docsJSON;

  docs: OpenApi;

  routes!: IRoute[];

  url!: string;

  static current_build;

  static server_ip = process.env.SERVER_IP;

  @Get()
  @Endpoint()
  @Summary("Список маршрутов")
  @Responses({
    status: 200,
    description: "Список маршрутов",
    isArray: true,
    schema: RouteElement,
  })
  static Index(@This() { routes }: RootRoute): IRoute[] {
    const result = [];
    routes.forEach((route) => {
      const operationId = getDisplayName(route.constructor, route.property);
      result.push(Object.assign(route, { operationId }));
    });
    return result;
  }

  @Get("/docs.json")
  @Use(BasicAuth.Required)
  @Summary("OpenApi JSON schema")
  @Responses({
    status: 200,
    description: "Документация OpenAPI",
    schema: { type: "string" },
  })
  static JSONDocs(@This() { docs }: RootRoute) {
    // сохраним документацию в контексте класса, чтобы не генерировать ее заново при повторных вызовах
    if (!this.docsJSON) {
      this.docsJSON = docs.toJSON();
    }
    return this.docsJSON;
  }

  @Get("/swagger.html")
  @Use(BasicAuth.Required)
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
