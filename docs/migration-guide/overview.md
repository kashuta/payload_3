---
title: Руководство по миграции с 2.0 на 3.0
label: Руководство по миграции с 2.0 на 3.0
order: 10
desc: Руководство по обновлению проектов Payload 2.x до версии 3.0.
keywords: local api, config, configuration, documentation, Content Management System, cms, headless, javascript, node, react
---

# Руководство по миграции с Payload 2.0 на 3.0

Payload 3.0 полностью переносит Панель Администратора с одностраничного приложения на React Router на Next.js App Router с полной поддержкой React Server Components. Это изменение полностью отделяет "ядро" Payload от его слоев рендеринга и HTTP, делая его по-настоящему Node-безопасным и портативным.

Если вы обновляетесь с _предыдущей бета-версии_, пожалуйста, обратитесь к разделу [Обновление с предыдущей бета-версии](#upgrade-from-previous-beta).

## Что изменилось?

Основная логика и принципы Payload остаются теми же от 2.0 до 3.0, при этом большинство изменений затрагивают конкретно HTTP-слой и Панель Администратора, которая теперь построена на Next.js. С этим изменением всё ваше приложение может быть обслуживаться в рамках одного репозитория, с конечными точками Payload, теперь открытыми непосредственно в вашем собственном приложении Next.js, рядом с вашим фронтендом. Payload остаётся headless, вы всё равно сможете использовать его полностью безголовым способом, как сейчас, с Sveltekit и т.д. Все API Payload остаются точно такими же (с несколькими новыми функциями), а конфигурация Payload в целом та же, с подробностями о критических изменениях, описанными ниже.

### Содержание

Все критические изменения перечислены ниже. Если вы столкнулись с изменениями, которые явно не указаны здесь, пожалуйста, рассмотрите возможность внести вклад в эту документацию, отправив PR.

- [Установка](#installation)
- [Критические изменения](#breaking-changes)
- [Пользовательские компоненты](#custom-components)
- [Конечные точки](#endpoints)
- [React хуки](#react-hooks)
- [Типы](#types)
- [Адаптеры электронной почты](#email-adapters)
- [Плагины](#plugins)
- [Обновление с предыдущей бета-версии](#upgrade-from-previous-beta)

## Установка

Payload 3.0 требует набор автоматически генерируемых файлов, которые вам нужно будет добавить в ваш существующий проект. Самый простой способ получения этих файлов - инициализировать новый проект через `create-payload-app`, затем заменить предоставленную конфигурацию Payload на вашу собственную.

```bash
  npx create-payload-app
```

Для получения более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/getting-started/installation).

1. **Установите новые зависимости Payload, Next.js и React**:

    Обратитесь к файлу package.json, созданному в create-payload-app, включая peerDependencies, devDependencies и dependencies. Основной пакет и плагины требуют синхронизации всех версий. Ранее, в 2.x было возможно использовать последнюю версию payload 2.x со старой версией db-mongodb, например. Теперь это невозможно.

    ```bash
      pnpm i next react react-dom payload @payloadcms/ui @payloadcms/next
    ```

    Также установите другие пакеты @payloadcms, специфичные для используемых вами плагинов и адаптеров. В зависимости от вашего проекта, они могут включать:
      - @payloadcms/db-mongodb
      - @payloadcms/db-postgres
      - @payloadcms/richtext-slate
      - @payloadcms/richtext-lexical
      - @payloadcms/plugin-form-builder
      - @payloadcms/plugin-nested-docs
      - @payloadcms/plugin-redirects
      - @payloadcms/plugin-relationship
      - @payloadcms/plugin-search
      - @payloadcms/plugin-sentry
      - @payloadcms/plugin-seo
      - @payloadcms/plugin-stripe
      - @payloadcms/plugin-cloud-storage - Читайте [Подробнее](#@payloadcms/plugin-cloud-storage).

1. Удалите устаревшие пакеты:

    ```bash
    pnpm remove express nodemon @payloadcms/bundler-webpack @payloadcms/bundler-vite
    ```

1. Миграции адаптеров баз данных

    _Если у вас есть существующие данные_ и вы используете адаптеры MongoDB или Postgres, вам нужно будет запустить миграции баз данных, чтобы убедиться, что схема вашей базы данных обновлена.

    - [postgres](https://github.com/payloadcms/payload/releases/tag/v3.0.0-beta.39)
    - [mongodb](https://github.com/payloadcms/payload/releases/tag/v3.0.0-beta.131)

2. Для пользователей Payload Cloud плагин изменился.

    Удалите старый пакет:

    ```bash
    pnpm remove @payloadcms/plugin-cloud
    ```

    Установите новый пакет:

    ```bash
    pnpm i @payloadcms/payload-cloud
    ```

    ```diff
    // payload.config.ts
    - import { payloadCloud } from '@payloadcms/plugin-cloud'
    + import { payloadCloudPlugin } from '@payloadcms/payload-cloud'

    buildConfig({
      // ...
      plugins: [
    -   payloadCloud()
    +   payloadCloudPlugin()
      ]
    })
    ```

3. **Опционально** зависимость sharp

   Если у вас есть коллекции с загрузками, которые используют `formatOptions`, `imageSizes` или `resizeOptions` — Payload ожидает наличия установленного `sharp`. В 2.0 эта зависимость устанавливалась для вас. Теперь она устанавливается только при необходимости. Если у вас установлены любые из этих опций, вам нужно будет установить `sharp` и добавить его в ваш payload.config.ts:

    ```bash
    pnpm i sharp
    ```

    ```diff
    // payload.config.ts
    import sharp from 'sharp'
    buildConfig({
    // ...
    +   sharp,
    })
    ```

## Критические изменения

1. Удалите свойство `admin.bundler` из вашей конфигурации Payload. Payload больше не бандлит Панель Администратора. Вместо этого мы полагаемся непосредственно на Next.js для бандлинга.

    ```diff
    // payload.config.ts
    - import { webpackBundler } from '@payloadcms/bundler-webpack'

    buildConfig({
      // ...
      admin: {
        // ...
    -   bundler: webpackBundler(),
      }
    })
    ```

    Это также означает, что пакеты `@payloadcms/bundler-webpack` и `@payloadcms/bundler-vite` устарели. Вы можете полностью удалить их из вашего проекта, удалив их из файла `package.json` и заново запустив процесс установки вашего пакетного менеджера, т.е. `pnpm i`.

1. Добавьте свойство `secret` в вашу конфигурацию Payload. Раньше оно устанавливалось в функции `payload.init()` вашего файла `server.ts`. Вместо этого переместите его в `payload.config.ts`:

    ```diff
    // payload.config.ts

    buildConfig({
      // ...
    + secret: process.env.PAYLOAD_SECRET
    })
    ```

1. Переменные среды с префиксом `PAYLOAD_PUBLIC` больше не будут доступны на клиенте. Чтобы получить к ним доступ на клиенте, теперь они должны иметь префикс `NEXT_PUBLIC` вместо этого.

    ```diff
    'use client'
    - const var = process.env.PAYLOAD_PUBLIC_MY_ENV_VAR
    + const var = process.env.NEXT_PUBLIC_MY_ENV_VAR
    ```

    Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/configuration/environment-vars).

1. Объект `req` раньше расширял [Express Request](https://expressjs.com/), но теперь расширяет [Web Request](https://developer.mozilla.org/en-US/docs/Web/API/Request). Возможно, вам потребуется соответствующим образом обновить ваш код, чтобы отразить это изменение. Например:

    ```diff
    - req.headers['content-type']
    + req.headers.get('content-type')
    ```

1. Свойства `admin.css` и `admin.scss` в конфигурации Payload были удалены.

    ```diff
    // payload.config.ts

    buildConfig({
      // ...
      admin: {
        // ...
    -   css: '',
    -   scss: ''
      }
    })
    ```

    Для миграции выберите один из следующих вариантов:

    1. Для большинства случаев использования вы можете просто настроить файл, расположенный в `(payload)/custom.scss`. Здесь вы можете импортировать или добавить свои собственные стили, например, для Tailwind.
    1. Для авторов плагинов вы можете использовать Custom Provider в `admin.components.providers` для импорта вашей таблицы стилей:

        ```tsx
        // payload.config.js

        //...
        admin: {
          components: {
            providers: [
              MyProvider: './providers/MyProvider.tsx'
            ]
          }
        },
        //...

        // providers/MyProvider.tsx

        'use client'
        import React from 'react'
        import './globals.css'

        export const MyProvider: React.FC<{children?: any}= ({ children }) ={
          return (
            <React.fragment>
              {children}
            </React.fragment>
          )
        }
        ```

1. Свойство `admin.indexHTML` было удалено. Удалите его из вашей конфигурации Payload.

    ```diff
    // payload.config.ts

    buildConfig({
      // ...
      admin: {
        // ...
    -   indexHTML: ''
      }
    })
    ```

1. Свойство `collection.admin.hooks` было удалено. Вместо этого используйте новый хук уровня поля `beforeDuplicate`, который принимает обычные аргументы хука поля.

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      admin: {
        hooks: {
    -     beforeDuplicate: ({ data }) => {
    -       return {
    -         ...data,
    -         title: `${data.title}-duplicate`
    -       }
    -     }
        }
      },
      fields: [
        {
          name: 'title',
          type: 'text',
          hooks: {
    +      beforeDuplicate: [
    +        ({ data }) => `${data.title}-duplicate`
    +      ],
          },
        },
      ],
    }
    ```

1. К полям с `unique: true` теперь автоматически будет добавляться "- Copy" через новые хуки поля `admin.beforeDuplicate` (см. предыдущий пункт).

1. Свойство `upload.staticDir` теперь должно быть абсолютным путем. Раньше оно пыталось использовать местоположение конфигурации Payload и объединять относительный путь, установленный для staticDir.

    ```diff
    // collections/Media.ts
    import type { CollectionConfig } from 'payload'
    import path from 'path'
    + import { fileURLToPath } from 'url'

    + const filename = fileURLToPath(import.meta.url)
    + const dirname = path.dirname(filename)

    export const MediaCollection: CollectionConfig = {
      slug: 'media',
      upload: {
    -   staticDir: path.resolve(__dirname, './uploads'),
    +   staticDir: path.resolve(dirname, '../uploads'),
      },
    }
    ```

1. Свойство `upload.staticURL` было удалено. Если вы использовали этот формат URL при использовании внешнего провайдера, вы можете использовать функции `generateFileURL` для выполнения той же задачи.

    ```diff
    // collections/Media.ts
    import type { CollectionConfig } from 'payload'

    export const MediaCollection: CollectionConfig = {
      slug: 'media',
      upload: {
    -   staticURL: '',
      },
    }
    ```

1. Свойство `admin.favicon` теперь называется `admin.icons`, и типы изменились:

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    const config = buildConfig({
      // ...
      admin: {
    -   favicon: 'path-to-favicon.svg',
    +   icons: [{
    +     path: 'path-to-favicon.svg',
    +     sizes: '32x32'
    +   }]
      }
    })
    ```

    Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/admin/metadata#icons).

1. Свойство `admin.meta.ogImage` было заменено на `admin.meta.openGraph.images`:

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    const config = buildConfig({
      // ...
      admin: {
        meta: {
    -     ogImage: '',
    +     openGraph: {
    +       images: []
    +     }
        }
      }
    })
    ```

    Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/admin/metadata#open-graph).

1. Аргументы функции `admin.livePreview.url` изменились. Она больше не получает аргумент `documentInfo`, а вместо этого теперь имеет `collectionConfig` и `globalConfig`.

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    export default buildConfig({
      // ...
      admin: {
        // ...
        livePreview: ({
    -     documentInfo,
    +     collectionConfig,
    +     globalConfig
        }) => ''
      }
    })
    ```

1. Свойства `admin.logoutRoute` и `admin.inactivityRoute` были объединены в единое свойство `admin.routes`. Для миграции просто переместите эти два ключа следующим образом:

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    const config = buildConfig({
      // ...
      admin: {
    -   logoutRoute: '/custom-logout',
    +   inactivityRoute: '/custom-inactivity'
    +   routes: {
    +     logout: '/custom-logout',
    +     inactivity: '/custom-inactivity'
    +   }
      }
    })
    ```

1. Свойство `custom` в конфигурации Payload, т.е. в Collections, Globals и Fields, теперь доступно **только на сервере** и **не** будет отображаться в клиентской сборке. Чтобы добавить пользовательские свойства в клиентскую сборку, используйте новое свойство `admin.custom`, которое будет доступно _как_ на сервере, _так и_ на клиенте.

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    export default buildConfig({
      custom: {
        someProperty: 'My Server Prop' // Теперь только на сервере!
      },
      admin: {
    +   custom: {
    +     name: 'My Client Prop' // Доступно на сервере И клиенте
    +   }
      },
    })
    ```

1. `hooks.afterError` теперь является массивом функций вместо одной функции. Аргументы также были расширены. Прочитайте [Подробнее](https://payloadcms.com/docs/hooks/overview#root-hooks).

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'

    export default buildConfig({
      hooks: {
    -   afterError: async ({ error }) => {
    +   afterError: [
    +     async ({ error, req, res }) => {
    +       // ...
    +     }
    +   ]
      }
    })
    ```
1. Директория `./src/public` теперь находится непосредственно на корневом уровне `./public` [смотрите документацию Next.js для деталей](https://nextjs.org/docs/pages/building-your-application/optimizing/static-assets)

1.  Payload теперь автоматически удаляет свойство `localized: true` из подполей, если родитель локализован, так как это избыточно и излишне. Если у вас есть существующие данные в этой структуре и вы хотите отключить это поведение, вам нужно включить флаг `allowLocalizedWithinLocalized` в вашем payload.config [подробнее в документации](https://payloadcms.com/docs/configuration/overview#compatibility-flags), или создать скрипт миграции, который выравнивает ваши данные.
Пример MongoDB для ссылки в макете страницы.

    ```diff
    - layout.columns.en.link.en.type.en
    + layout.columns.en.link.type
    ```
## Пользовательские компоненты

1. Все компоненты React Payload были перемещены из пакета `payload` в `@payloadcms/ui`. Если вы ранее импортировали компоненты в ваше приложение из пакета `payload`, например, для создания пользовательских компонентов, вам нужно будет изменить пути импорта:

    ```diff
    - import { TextField, useField, etc. } from 'payload'
    + import { TextField, useField, etc. } from '@payloadcms/ui'
    ```
   *Примечание: для краткости здесь перечислены _не все_ модули*

1. Все пользовательские компоненты теперь определяются как _пути к файлам_, а не как прямые импорты. Если вы используете пользовательские компоненты в вашей конфигурации Payload, удалите импортированный модуль и укажите вместо него путь к файлу:

    ```diff
    import { buildConfig } from 'payload'
    - import { MyComponent } from './src/components/Logout'

    const config = buildConfig({
      // ...
      admin: {
        components: {
          logout: {
    -       Button: MyComponent,
    +       Button: '/src/components/Logout#MyComponent'
          }
        }
      },
    })
    ```

   Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/custom-components/overview#component-paths).

1. Все пользовательские компоненты теперь рендерятся на сервере по умолчанию и поэтому не могут использовать состояние или хуки напрямую. Если вы используете пользовательские компоненты в вашем приложении, которые требуют состояние или хуки, добавьте директиву `'use client'` в начало файла.

    ```diff
    // components/MyClientComponent.tsx
    + 'use client'
    import React, { useState } from 'react'

    export const MyClientComponent = () => {
      const [state, setState] = useState()

      return (
        <div>
          {state}
        </div>
      )
    }
    ```

   Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/custom-components/overview#client-components).

1. Свойство `admin.description` внутри Collection, Globals и Fields больше не принимает компонент React. Вместо этого вы должны определить его как пользовательский компонент.

    1. Для Collections используйте ключ `admin.components.edit.Description`:

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'
    - import { MyCustomDescription } from '../components/MyCustomDescription'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      admin: {
    -    description: MyCustomDescription,
    +    components: {
    +      edit: {
    +        Description: 'path/to/MyCustomDescription'
    +      }
    +    }
      }
    }
    ```

    2. Для Globals используйте ключ `admin.components.elements.Description`:

    ```diff
    // globals/Site.ts
    import type { GlobalConfig } from 'payload'
    - import { MyCustomDescription } from '../components/MyCustomDescription'

    export const SiteGlobal: GlobalConfig = {
      slug: 'site',
      admin: {
    -    description: MyCustomDescription,
    +    components: {
    +      elements: {
    +        Description: 'path/to/MyCustomDescription'
    +      }
    +    }
      }
    }
    ```

    3. Для Fields используйте ключ `admin.components.Description`:

    ```diff
    // fields/MyField.ts
    import type { FieldConfig } from 'payload'
    - import { MyCustomDescription } from '../components/MyCustomDescription'

    export const MyField: FieldConfig = {
      type: 'text',
      admin: {
    -    description: MyCustomDescription,
    +    components: {
    +      Description: 'path/to/MyCustomDescription'
    +    }
      }
    }
    ```

1. Метки строк Array Field и метка Collapsible Field теперь _только_ принимают компонент React и больше не принимают простую строку или запись:

    ```diff
    // file: Collection.tsx
    import type { CollectionConfig } from 'payload'
    - import { MyCustomRowLabel } from './components/MyCustomRowLabel.tsx'

    export const MyCollection: CollectionConfig = {
      slug: 'my-collection',
      fields: [
        {
          name: 'my-array',
          type: 'array',
          admin: {
            components: {
    -         RowLabel: 'My Array Row Label,
    +         RowLabel: './components/RowLabel.ts'
            }
          },
          fields: [...]
        },
        {
          name: 'my-collapsible',
          type: 'collapsible',
          admin: {
            components: {
    -         Label: 'My Collapsible Label',
    +         Label: './components/RowLabel.ts'
            }
          },
          fields: [...]
        }
      ]
    }
    ```

1. Все ключи представлений по умолчанию теперь в camelcase:

   Например, для корневых представлений:

    ```diff
    // file: payload.config.ts

    import { buildConfig } from 'payload'

    export default buildConfig({
    admin: {
      views: {
    -    Account: ...
    +    account: ...
      }
    })
    ```

   Или представления документов:

    ```diff
    // file: Collection.tsx

    import type { CollectionConfig } from 'payload'

    export const MyCollection: CollectionConfig = {
      slug: 'my-collection',
      admin: {
        views: {
    -     Edit: {
    -       Default: ...
    -     }
    +     edit: {
    +       default: ...
    +     }
        }
      }
    }
    ```

1. Пользовательские представления в конфигурации больше не принимают компоненты React напрямую, вместо этого вы должны использовать их свойство `Component`:

    ```diff
    // file: Collection.tsx
    import type { CollectionConfig } from 'payload'
    - import { MyCustomView } from './components/MyCustomView.tsx'

    export const MyCollection: CollectionConfig = {
      slug: 'my-collection',
      admin: {
        views: {
    -     Edit: MyCustomView
    +     edit: {
    +       Component: './components/MyCustomView.tsx'
    +     }
        }
      }
    }
    ```

   Это также означает, что пользовательские корневые представления больше не определяются по ключу `edit`. Вместо этого используйте новый ключ `views.root`:

    ```diff
    // file: Collection.tsx
    import type { CollectionConfig } from 'payload'
    - import { MyCustomRootView } from './components/MyCustomRootView.tsx'

    export const MyCollection: CollectionConfig = {
      slug: 'my-collection',
      admin: {
        views: {
    -     Edit: MyCustomRootView
          edit: {
    +       root: {
    +         Component: './components/MyCustomRootView.tsx'
    +       }
          }
        }
      }
    }
    ```

1. Функции `href` и `isActive` на вкладках представления больше не включают аргументы `match` или `location`. Это свойство специфично для React Router, а не для Next.js. Если вам нужно выполнить сопоставление URL, аналогичное этому, используйте собственную вкладку, которая запускает некоторые хуки, например, `usePathname()`, и запустите ее через ваши собственные служебные функции:

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      admin: {
        components: {
          views: {
    -        Edit: {
    -          Tab: {
    -            isActive: ({ href, location, match }) => true,
    -            href: ({ href, location, match }) => ''
    -          },
    -       },
    +       edit: {
    +         tab: {
    +           isActive: ({ href }) => true,
    +           href: ({ href }) => '' // или используйте пользовательский компонент (см. ниже)
    +           // Component: './path/to/CustomComponent.tsx'
    +         }
    +       },
          },
        },
      },
    }
    ```

1. Свойство `admin.components.views[key].Tab.pillLabel` было заменено на `admin.components.views[key].tab.Pill`:

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      admin: {
        components: {
    -     views: {
    -       Edit: {
    -         Tab: {
    -           pillLabel: 'Hello, world!',
    -         },
    -       },
    +       edit: {
    +         tab: {
    +           Pill: './path/to/CustomPill.tsx',
    +         }
    +       },
          },
        },
      },
    }
    ```

1. `react-i18n` был удален, компонент `Trans` из `react-i18n` был заменен решением, предоставляемым Payload:

    ```diff
    'use client'
    - import { Trans } from "react-i18n"
    + import { Translation } from "@payloadcms/ui"

    // Пример строки для перевода:
    // "loggedInChangePassword": "To change your password, go to your <0>account</0> and edit your password there."

    export const MyComponent = () => {
      return (
    -     <Trans i18nKey="loggedInChangePassword" t={t}>
    -       <Link to={`${admin}/account`}>account</Link>
    -     </Trans>

    +     <Translation
    +       t={t}
    +       i18nKey="authentication:loggedInChangePassword"
    +       elements={{
    +         '0': ({ children }) => <Link href={`${admin}/account`} children={children} />,
    +       }}
    +     />
      )
    }
    ```

## Конечные точки

1. Все обработчики конечных точек изменились. Аргументы больше не включают `res` и `next`, а тип возврата теперь ожидает действительный HTTP [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) вместо `res.json`, `res.send` и т.д.:

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      endpoints: [
    -   {
    -     path: '/whoami/:parameter',
    -     method: 'post',
    -     handler: (req, res) => {
    -       res.json({
    -         parameter: req.params.parameter,
    -         name: req.body.name,
    -         age: req.body.age,
    -       })
    -     }
    -   },
    +   {
    +     path: '/whoami/:parameter',
    +     method: 'post',
    +     handler: (req) => {
    +       return Response.json({
    +         parameter: req.routeParams.parameter,
    +         // ^^ `params` теперь `routeParams`
    +         name: req.data.name,
    +         age: req.data.age,
    +       })
    +     }
    +   }
      ]
    }
    ```

1. Обработчики конечных точек больше не разрешают `data`, `locale` или `fallbackLocale` для вас в запросе. Вместо этого вы должны разрешить их самостоятельно или использовать утилиты, предоставляемые Payload:

    ```diff
    // collections/Posts.ts
    import type { CollectionConfig } from 'payload'
    + import { addDataAndFileToRequest } from '@payloadcms/next/utilities'
    + import { addLocalesToRequest } from '@payloadcms/next/utilities'

    export const PostsCollection: CollectionConfig = {
      slug: 'posts',
      endpoints: [
    -   {
    -     path: '/whoami/:parameter',
    -     method: 'post',
    -     handler: async (req) => {
    -       return Response.json({
    -         name: req.data.name, // data будет неопределен
    -       })
    -     }
    -   },
    +   {
    +     path: '/whoami/:parameter',
    +     method: 'post',
    +     handler: async (req) => {
    +       // изменяет req, должен быть ожидаемым
    +       await addDataAndFileToRequest(req)
    +       await addLocalesToRequest(req)
    +
    +       return Response.json({
    +         name: req.data.name, // теперь data доступен
    +    	    fallbackLocale: req.fallbackLocale,
    +         locale: req.locale,
    +       })
    +     }
    +   }
      ]
    }
    ```

## React хуки

1. Хук `useTitle` был объединен с хуком `useDocumentInfo`. Вместо этого вы можете получить title напрямую из контекста информации о документе:

    ```diff
    'use client'
    - import { useTitle } from 'payload'
    + import { useDocumentInfo } from '@payloadcms/ui'

    export const MyComponent = () => {
    - const title = useTitle()
    + const { title } = useDocumentInfo()

      // ...
    }
    ```

1. Хук `useDocumentInfo` больше не возвращает `collection` или `global`. Вместо этого передаются различные свойства конфигурации, такие как `collectionSlug` и `globalSlug`. Вы можете использовать их для доступа к клиентской конфигурации, если необходимо, через хук `useConfig` (см. следующий пункт).

    ```diff
    'use client'
    import { useDocumentInfo } from '@payloadcms/ui'

    export const MyComponent = () => {
      const {
    -   collection,
    -   global,
    +   collectionSlug,
    +   globalSlug
      } = useDocumentInfo()

      // ...
    }
    ```

1. Хук `useConfig` теперь возвращает `ClientConfig`, а не `SanitizedConfig`. Это потому что сама конфигурация не сериализуема и поэтому не может быть передана на клиент. Это означает, что все несериализуемые свойства были исключены из клиентской конфигурации, такие как `db`, `bundler` и т.д.

    ```diff
    'use client'
    - import { useConfig } from 'payload'
    + import { useConfig } from '@payloadcms/ui'

    export const MyComponent = () => {
    - const config = useConfig() // раньше был 'SanitizedConfig'
    + const { config } = useConfig() // теперь это 'ClientConfig'

      // ...
    }
    ```

   Для более подробной информации, смотрите [Документацию](https://payloadcms.com/docs/admin/custom-components/overview#accessing-the-payload-config).

1. Хук `useCollapsible` претерпел небольшие изменения названий свойств. `collapsed` теперь называется `isCollapsed`, а `withinCollapsible` теперь называется `isWithinCollapsible`.

    ```diff
    'use client'
    import { useCollapsible } from '@payloadcms/ui'

    export const MyComponent = () => {
    - const { collapsed, withinCollapsible } = useCollapsible()
    + const { isCollapsed, isWithinCollapsible } = useCollapsible()
    }
    ```

1. Хук `useTranslation` больше не принимает никаких опций, любые переводы, использующие сокращенные средства доступа, должны использовать полный `group:key`

    ```diff
    'use client'
    - import { useTranslation } from 'payload'
    + import { useTranslation } from '@payloadcms/ui'

    export const MyComponent = () => {
    - const { i18n, t } = useTranslation('general')
    + const { i18n, t } = useTranslation()

    - return <p>{t('cancel')}</p>
    + return <p>{t('general:cancel')}</p>
    }
    ```

## Типы

1. Тип `Fields` был переименован в `FormState` для улучшения семантики. Если вы ранее импортировали этот тип в вашем собственном приложении, просто измените имя импорта:

    ```diff
    - import type { Fields } from 'payload'
    + import type { FormState } from 'payload'
    ```

1. Типы `BlockField` и связанные с ним были переименованы в `BlocksField` для семантической точности.

    ```diff
    - import type { BlockField, BlockFieldProps } from 'payload'
    + import type { BlocksField, BlocksFieldProps } from 'payload'
    ```

## Адаптеры электронной почты

Функциональность электронной почты была абстрагирована в адаптеры электронной почты.

- Вся существующая функциональность nodemailer была абстрагирована в пакет `@payloadcms/email-nodemailer`
- Больше не конфигурируется с ethereal.email по умолчанию.
- Возможность передавать email в функцию `init` была удалена.
- Предупреждение будет выдано при запуске, если email не настроен. Любой вызов `sendEmail` просто выведет в лог адрес To и тему.
- Адаптер Resend теперь также доступен через пакет `@payloadcms/email-resend`.

### Если вы использовали конфигурацию электронной почты по умолчанию в 2.0 (nodemailer):

```tsx
// ❌ До

// через payload.init
payload.init({
  email: {
    transport: someNodemailerTransport
    fromName: 'hello',
    fromAddress: 'hello@example.com',
  },
})
// или через email в payload.config.ts
export default buildConfig({
  email: {
    transport: someNodemailerTransport
    fromName: 'hello',
    fromAddress: 'hello@example.com',
  },
})

// ✅ После

// Использование нового пакета адаптера nodemailer

import { nodemailerAdapter } from '@payloadcms/email-nodemailer'

export default buildConfig({
  email: nodemailerAdapter() // Это будет старая функциональность ethereal.email
})

// или передайте transport

export default buildConfig({
  email: nodemailerAdapter({
    defaultFromAddress: 'info@payloadcms.com',
    defaultFromName: 'Payload',
    transport: await nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: 587,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    })
  })
})
```

### Удаление ограничения скорости

- Теперь доступно только при использовании пользовательского сервера и использовании express или подобного

## Плагины

1. *Все* плагины были стандартизированы для использования _именованных экспортов_ (в отличие от экспортов по умолчанию). Большинство также имеют суффикс `Plugin`, чтобы было ясно, что импортируется.

    ```diff
    - import seo from '@payloadcms/plugin-seo'
    + import { seoPlugin } from '@payloadcms/plugin-seo'

    - import stripePlugin from '@payloadcms/plugin-stripe'
    + import { stripePlugin } from '@payloadcms/plugin-stripe'

    // и так далее для каждого плагина
    ```

## `@payloadcms/plugin-cloud-storage`

- Адаптеры, экспортируемые из пакета `@payloadcms/plugin-cloud-storage` (например, `@payloadcms/plugin-cloud-storage/s3`), были удалены.
- Были созданы новые *отдельные* пакеты для каждого из существующих адаптеров. Пожалуйста, ознакомьтесь с документацией для того, который вы используете.
- `@payloadcms/plugin-cloud-storage` по-прежнему полностью поддерживается, но должен использоваться только если вы предоставляете пользовательский адаптер, который не имеет специального пакета.
- Если вы создали пользовательский адаптер, тип теперь должен предоставить свойство `name`.

| Сервис              | Пакет                                                                        |
| -------------------- | ---------------------------------------------------------------------------- |
| Vercel Blob          | https://github.com/payloadcms/payload/tree/main/packages/storage-vercel-blob |
| AWS S3               | https://github.com/payloadcms/payload/tree/main/packages/storage-s3          |
| Azure                | https://github.com/payloadcms/payload/tree/main/packages/storage-azure       |
| Google Cloud Storage | https://github.com/payloadcms/payload/tree/main/packages/storage-gcs         |

```tsx
// ❌ До (требовались пир-зависимости в зависимости от адаптера)

import { cloudStorage } from '@payloadcms/plugin-cloud-storage'
import { s3Adapter } from '@payloadcms/plugin-cloud-storage/s3'

plugins: [
    cloudStorage({
      collections: {
        media: {
          adapter: s3Adapter({
            bucket: process.env.S3_BUCKET,
            config: {
              credentials: {
                accessKeyId: process.env.S3_ACCESS_KEY_ID,
                secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
              },
              region: process.env.S3_REGION,
            },
          }),
        },
      },
    }),
  ],

 // ✅ После

 import { s3Storage } from '@payloadcms/storage-s3'

 plugins: [
    s3Storage({
      collections: {
        media: true,
      },
      bucket: process.env.S3_BUCKET,
      config: {
        credentials: {
          accessKeyId: process.env.S3_ACCESS_KEY_ID,
          secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
        },
        region: process.env.S3_REGION,
      },
    }),
  ],
```

## `@payloadcms/plugin-form-builder`

1. Переопределения полей для коллекций форм и отправок форм теперь принимают функцию с `defaultFields` внутри аргументов вместо массива конфигурации

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'
    import { formBuilderPlugin } from '@payloadcms/plugin-form-builder'

    const config = buildConfig({
      // ...
      plugins: formBuilderPlugin({
    -   fields: [
    -     {
    -       name: 'custom',
    -       type: 'text',
    -     }
    -   ],
    +   fields: ({ defaultFields }) => {
    +     return [
    +       ...defaultFields,
    +       {
    +         name: 'custom',
    +         type: 'text',
    +       },
    +     ]
    +   }
      })
    })
    ```

## `@payloadcms/plugin-redirects`

1. Переопределения полей для коллекции перенаправлений теперь принимают функцию с `defaultFields` внутри аргументов вместо массива конфигурации

    ```diff
    // payload.config.ts
    import { buildConfig } from 'payload'
    import { redirectsPlugin } from '@payloadcms/plugin-redirects'

    const config = buildConfig({
      // ...
      plugins: redirectsPlugin({
    -   fields: [
    -     {
    -       name: 'custom',
    -       type: 'text',
    -     }
    -   ],
    +   fields: ({ defaultFields }) => {
    +     return [
    +       ...defaultFields,
    +       {
    +         name: 'custom',
    +         type: 'text',
    +       },
    +     ]
    +   }
      })
    })
    ```

## `@payloadcms/richtext-lexical`

Если у вас есть пользовательские функции для `@payloadcms/richtext-lexical`, вам нужно будет мигрировать ваш код на новый API. Прочитайте больше о новом API в [документации](https://payloadcms.com/docs/rich-text/building-custom-features).

## Зарезервированные имена полей

Payload резервирует определенные имена полей для внутреннего использования. Использование любого из следующих имен в ваших коллекциях или globals приведет к очистке этих полей из конфигурации, что может вызвать ошибки развертывания. Убедитесь, что любые конфликтующие поля переименованы перед миграцией.

### Общие зарезервированные имена

- `file`
- `_id` (только MongoDB)
- `__v` (только MongoDB)

**Важное примечание**: Рекомендуется избегать использования имен полей с префиксом подчеркивания (`_`), если это явно не требуется плагином. Payload использует этот префикс для внутренних столбцов, что может привести к конфликтам в определенных условиях SQL. Следующие примеры являются зарезервированными внутренними столбцами (этот список не является исчерпывающим, и другие внутренние поля также могут применяться):

- `_order`
- `_path`
- `_uuid`
- `_parent_id`
- `_locale`

### Зарезервированные имена, связанные с аутентификацией

Эти имена ограничены, если ваша коллекция использует `auth: true` и не имеет `disableAuthStrategy: true`:
- `salt`
- `hash`
- `apiKey` (когда включено `auth.useAPIKey: true`)
- `useAPIKey` (когда включено `auth.useAPIKey: true`)
- `resetPasswordToken`
- `resetPasswordExpiration`
- `password`
- `email`
- `username`

### Зарезервированные имена, связанные с загрузкой

Эти имена применяются, если ваша коллекция имеет настроенное `upload: true`:

- `filename`
- `mimetype`
- `filesize`
- `width`
- `height`
- `focalX`
- `focalY`
- `url`
- `thumbnailURL`

Если настроено `imageSizes`, также зарезервировано следующее:

- `sizes`

Если любое из этих имен найдено в полях вашей коллекции / global, обновите их перед миграцией, чтобы избежать непредвиденных проблем.

## Обновление с предыдущей бета-версии

Обратитесь к этому [сайту, созданному сообществом](https://payload-releases-filter.vercel.app/?version=3&from=152429656&to=188243150&sort=asc&breaking=on). Установите вашу версию, отсортируйте по самым старым сначала, включите только критические изменения.

Затем пройдитесь по каждому из критических изменений и внесите корректировки. Вы можете опционально обратиться к [пустому шаблону](https://github.com/payloadcms/payload/tree/main/templates/blank) для ознакомления с тем, как всё должно выглядеть в итоге.
