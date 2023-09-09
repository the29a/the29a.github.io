---
type: post
title: "Подглядываем через Sourcemaps"
date: 2023-09-09T03:52:36+03:00
showTableOfContents: true
draft: false
tags: ["security", "hacking", "web"]
---

## Зачем нужны sourcemaps в JavaScript?
Файлы Source-map полезны при отладке минифицированных JavaScript-файлов. 
Они работают путем сопоставления минифицированного кода с его неминифицированной версией. После этого можно отлаживать неминифицированный Javascript как обычно.

## Как можно найти файл JS map?
Burp, DevTools браузера.
В конец минифицированного файла добавляется конструкция вроде этой:
```js
//# sourceMappingURL=main-1337666.js.map
```
Но в идеале никак. Оно должно быть в условном Sentry.

К слову о Sentry. Их мы и возьмем как пример.

## Что и где?
![](../../static/20230909030605.png)
В примере на скриншоте: 
- Множество минифицированных  JS-файлов
- Код минифицированного js-файла
- Ссылка на sourcemap-файл

Возьмем как пример файл `f10687e-943b37fa375b2046728e.js`.
В сжатом файле будет строчка со ссылкой на sourcemap:
```js
//# sourceMappingURL=f10687e-943b37fa375b2046728e.js.map
```

## Чем получить исходники?
Мы можем запустить curl:
```bash
curl https://33fa1ur95-bcmasgak4.sentry.dev/2f10687e-943b37fa375b2046728e.js.map -o test.js
```
Но в ответ получим глазную боль.

![](../../static/20230909034129.png)
Мы можем посмотреть это всё в браузере, но это тоже веселым занятием не назовешь.

Чтобы получить что-то полезное из нашей находки воспользуемся [sourcemapper](https://github.com/denandz/sourcemapper):
```bash
sourcemapper -url https://33fa1ur95-bcmasgak4.sentry.dev/2f10687e-943b37fa375b2046728e.js.map -output sourcemaps
```

Можно собрать список, и забрать всё оптом.
Например так:
```bash
cat sourcemaps.list | xargs -I {} sourcemapper -url {} -output sourcemaps
```

Примечание:
То ли баг, то ли фича. Бывает ошибка `status !=200`. Приходится забирать sourcemap вне списка.


## Зачем это всё?
Мы получаем исходный код проекта, пусть и не весь. А это нам открывает кучу возможностей.
- Анализ кода на уязимость (вдруг XSS найдется)
- Захардкоженые API ключи
- Захардкоженые пароли
- Ссылки на панели администрирования
- Информация о API-эндпоинтах
- Множество разной дополнительной информации из комментариев в коде.

В теории, это всё можно автоматизировать (было бы интересно), с запуском semgrep, trufflehog и прочим.

P.S. Можете попробовать сдать им этот баг, но у них вечные беды с bug bounty, а если просто так им на почту написать, то даже спасибо не скажут.

---
## Reference  
[Github: sourcemapper](https://github.com/denandz/sourcemapper)  
[Github: Burp-SourceMapper](https://github.com/yg-ht/Burp-SourceMapper)  
[Github: source-mapper](https://github.com/portswigger/source-mapper)  
[Web.Dev: Source Maps](https://web.dev/source-maps/)  
[Habr: Время подключать исходники. Введение в Source Maps](https://habr.com/ru/articles/178743/)  
[Github: Source maps: languages, tools and other info](https://github.com/ryanseddon/source-map/wiki/Source-maps:-languages,-tools-and-other-info)  