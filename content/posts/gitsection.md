---
title: "Git дисекция (гитсекция)"
date: 2016-04-21T21:20:25+03:00
draft: false
description: Git е нещо, което ви дава крила (ахъм) ако познавате достатъчно добре. И все пак, командният интерфейс на Git е едно от най-объркващите неща, които може да ви се наложи да използвате.
---

Git е нещо, което ви дава крила (ахъм) ако познавате достатъчно добре.

И все пак, командният интерфейс на Git е едно от най-объркващите неща, които може да ви се наложи да използвате.
Лично аз използвам git от години, дори му написах [нещо като клонинг](https://github.com/georgyangelov/scv) за курсов проект,
но все още откривам нови опции и начини за работа с него.

## Git internals

Въпреки планината от опции, вътрешната работа на git не е нещо особено сложно. Това е пример за нещо изключително мощно,
реализирано чрез малко прости правила.

Бих казал, че идеята зад git е една от най-елегантните. Това е и което го прави толкова мощен инструмент.

Да направим дисекция на едно git хранилище. Сигурно сте забелязали, че единствената разлика между обикновена папка и такава,
съдържаща git хранилище, е скритата папка `.git`. Да видим какво има вътре.

    $ tree .git
    .git
    ├── COMMIT_EDITMSG
    ├── FETCH_HEAD
    ├── HEAD
    ├── ORIG_HEAD
    ├── config
    ├── description
    ├── hooks
    ├── index
    ├── info
    │   ├── exclude
    │   └── refs
    ├── objects
    │   ├── 00
    │   │   ├── 294350c1d9d43cf0867a9caed1d7370d14725d
    │   │   ├── 2a542e73bc8c4fcc52876d8f3d3c491a15b3e7
    │   │   └── [...]
    │   ├── 01
    │   │   ├── 07226bfcfcbe58e07d4bf5345f54a07976d093
    │   │   ├── 430e9db98afb84858562e58b6ca91a38bf0776
    │   │   └── [...]
    │   ├── [...]
    │   ├── info
    │   │   └── packs
    │   └── pack
    │       ├── pack-108036f91354840150b3e0c230f6bf9aabbab4b7.idx
    │       ├── pack-108036f91354840150b3e0c230f6bf9aabbab4b7.pack
    │       ├── pack-56536e19282ac87714f1a5d47f66472e9205b350.idx
    │       ├── pack-56536e19282ac87714f1a5d47f66472e9205b350.pack
    │       ├── pack-98c2a424c1806f4aa7c6e7a58826f6da80e2ee8a.idx
    │       └── pack-98c2a424c1806f4aa7c6e7a58826f6da80e2ee8a.pack
    ├── packed-refs
    └── refs
        ├── heads
        │   ├── LL-1319
        │   ├── LL-1435-common-js-files
        │   ├── LL-1461-search-appearance
        │   ├── LL-1474-clean-up-codebase
        │   ├── LL-991-web-tests
        │   ├── internal-tasks
        │   ├── items-count-on-home
        │   ├── master
        │   ├── notification-preferences-report
        │   └── production
        ├── remotes
        │   └── origin
        │       ├── 1398_android_animations
        │       ├── HEAD
        │       ├── LL-1139-change-and-remove-profile-image
        │       └── [...]
        ├── stash
        └── tags
            └── web_v1.2.6

Очевидно съм изрязал някои неща от output-а `[...]`, за да не си запалите скрола.

### Обекти

Някои от тези неща изглеждат познато - например файловете с имена sha1 хешове в objects. Да се пробваме на някой:

    $ cat .git/objects/c1/d41c01e65da42fcf5815944cd7891240bacc5a
    x}U???6?U???rH
              ??-?R???i?lҢ???(??DKl(R%)
    )y??????p???Qm\M߼~??/h????bԶ           ???
                                e?AM???T????
    ??O?ۤӦ?&Na3?     ?:?:??X6稛p???^?V%??H?B'6?8???H?Rem???=}??UA??????m??-?m????????9?E :?	?,?F?
                                                                                                ????)??
                                                                                                        ?DM>VeѪ?x=F??8?B??gj?ĳ$???<??,?8???N?/??xq?F??}?xWCr???\q?G?P?Z]?ZV?[????צ??D???R?l?q??k??4
    ??B?????D?x??y??L?g?;H??B???طW?>B?2?GhmUG#!z??݁?#O&??#??h;?!2?ܧ?@????Ο/?:?$?U????m??Btq?H?̚?4r?ye?????r?3??e?o??0$???'?Y?HC'?Z??V???^?K#?Ɯ???????~P-lr9z???/E??P<6?cZ=?m??+?t?H??'<O??1?\?)f

    ?M/?%B?0?2???z?
    ?bO?`?,$?C?KBA?ٙ?c???E?!?
    5?7S??d[??~?"ʺ????~?g??o??<??L?L?ԄX3?E?z? iƛB?Iv?ή?lHc:S?>?]??!?jp?mr`%?SI?
                    5??\???;??Kʅ|OD???iz\@????L%??
    v?Bx??????y????-F??$,3??;???J~??37?????u ??

Добрее, очевидно файлът е компресиран. Лек гугъл сърч открива, че git обектите
[се компресират със zlib](http://stackoverflow.com/a/2028205/557600) и са в двоичен формат. За да си спестим време,
няма да се занимаваме да ги четем ръчно. Вместо това, ще използваме командата `git cat-file`, която може да декомпресира
обекти и да ги показва в текстов вид.

    $ git cat-file -t c1d41c01e65da42fcf5815944cd7891240bacc5a
    commit

Опцията `-t` ни казва какъв тип е git обектът, за който питаме. Явно това е commit. С `-p` можем да го накараме да ни покаже
съдържанието на файла като текст:

    $ git cat-file -p c1d41c01e65da42fcf5815944cd7891240bacc5a
    tree 7ebcc9c3341f68404ae314df71e0f2088c8b48bf
    parent e47b0466a1d9cdd829a0c30372d92ec72355c9e2
    author anonimen <anonimen@abv.bg> 1457628985 +0200
    committer anonimen <anonimen@abv.bg> 1457628985 +0200

    [channels_web] rename css class

Виждаме автора, съобщението на commit-а, timestamp-ове и някакви други хешове на обекти. Какво е `tree`?

    $ git cat-file -t 7ebcc9c3341f68404ae314df71e0f2088c8b48bf
    tree

    $ git cat-file -p 7ebcc9c3341f68404ae314df71e0f2088c8b48bf
    100644 blob 3c2bd6360bbe0d0a2547dc56038e30383555a600    .dockerignore
    100644 blob 8791029d1e95e50cb73093a2984cc883fb451b0a    .gitattributes
    100644 blob 07d725855ba7e9cafd25515c23f966d64b4713ee    .gitignore
    100644 blob 9b83a8cf9768e12b6d0ff410f35f9bc0c1562c57    .haml-lint.yml
    100644 blob cc61cab111c061e84c63f6b9666f2d2198404ded    .jscsrc
    100644 blob a1cd3560dcb46523b7fa754845436af105c08beb    .jshintignore
    100644 blob 18dea8b54bd5fb454a1bc72558e11252eb726296    .jshintrc
    100644 blob 83e16f80447460c937aeaa44a64d743b27863077    .rspec
    100755 blob edeb10a11d31a41c7796e938a20ddfc6b596886f    .rubocop.yml
    100644 blob e52c172cf7c39094c7146e89499feaedee50419b    .sass-lint.yml
    100644 blob e818b7925f10918da40e7190c052f943c79a2b84    Dockerfile
    100644 blob b7b2d8669141c84c7b4f32546098804fe3fa173c    Gemfile
    100644 blob 5f5144885188111460e5f21e90f1f590b20bda27    Gemfile.lock
    100644 blob 36e70693be1800aae4f24d8376a56ea33ff6246f    Makefile
    100644 blob dd4e97e22e159a585b20e21028f964827d5afa4e    README.rdoc
    100644 blob ba6b733dd2358d858f00445ebd91c214f0f5d2e5    Rakefile
    040000 tree ada42689bb07c448698269a63b3d7ad3b673c479    app
    040000 tree 19b9d6c8a39c3ecc9e985b34af88ab219e07df75    bin
    100644 blob de96225b78fd77beb0a8b99a0b8398dc8c6e30c3    config.ru
    040000 tree 74fdac0037df2d844293c870afec70b8bd354a88    config
    040000 tree 2d323cffa0a15fcfdbe6066f90a5cd08ef81f5bb    db
    040000 tree 41f4a03b5b9cdaa68e4d7b60ba9447abe62b50b8    lib
    040000 tree 29a422c19251aeaeb907175e9b3219a9bed6c616    log
    040000 tree 3d60a8b30140bc12accb7f4292b78f6915358543    mobile
    040000 tree f41e0797fa88469fd0440c5d0fad2660cee86000    public
    040000 tree f704e30906d0fca61ea0226a4fac210a4994763b    spec
    100644 blob a909cc7ee9d76bd43c738049cbdefb94959d0f45    test.yml
    040000 tree 56f75ae9a3cefff0197e1cc8e421bcee0639367b    vendor

Хм, това също изглежда познато - главната директория на проекта. Можем лесно да разпознаем кое са папки и кое - файлове.
Да видим какво ще каже за файла `.rspec`:

    $ git cat-file -t 83e16f80447460c937aeaa44a64d743b27863077
    blob

    $ git cat-file -p 83e16f80447460c937aeaa44a64d743b27863077
    --color
    --require spec_helper

Окей, това е съдържанието на файла. Явно `blob` са обекти, съдържащи файлове. Има логика. `tree` = папка, `blob` = файл.

### История

Да се върнем на commit-а. Освен tree виждаме и друг хеш на обект - parent.

    $ git cat-file -p e47b0466a1d9cdd829a0c30372d92ec72355c9e2
    tree ea4b4f4e4a6e0afb981e485c61a2cf170f8178a3
    parent 8d0fbf6294644e43a1c798d75f07a78d2b010650
    parent d4c93201bdeb23cbed4aaabc8bc180f2c6125f04
    author anonimen <anonimen@abv.bg>  1457626404 +0200
    committer anonimen <anonimen@abv.bg>  1457626404 +0200

    Merge branch 'channels_web' of github.com:.../... into channels_web

Ахаа! Още един commit! Всеки от вас знае какво е (едно)свързан списък. Е, историята на git е просто свързан списък от commit-и. Когато напишете git log, git просто хваща последния commit и проследява parent връзките.

Чакай малко! Този commit има двама родители. Свързаните списъци имат само по една връзка.

Добре де, поизлъгах. Всъщност историята на git е [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph). Ориентиран граф без цикли.

Та, този commit има два предходни, защото е merge commit. Неговата задача е да слее две разклонили се версии на кода. Някак интуитивно е, че би имал двама родители.

### ID-та

Вече знаем, че има поне три вида обекти в git - `commit`, `tree` и `blob`. Всеки един обект си има уникален идентификатор.
Този идентификатор е sha1 хешът на съдържанието на файла-обект. В декомпресирания си вид.

Още повече - git не допуска промяна на вече съществуващи обекти.
Ама това е супер яко! Това е едно от най-умните неща:

- Всеки обект има уникално ID
- Ако два обекта имат еднакво ID, значи са еднакви*. Това значи, че ако в два различни commit-а има един и същ файл,
  то той ще се пази от git само веднъж. Достатъчно е само да го хешираме и да проверим дали има обект с това име.
- Всеки обект-дърво съдържа ID-тата на нещата вътре. Значи ако се промени файл в обект дърво, то и ID-то на това дърво
  ще е различно. Така много лесно проверяваме дали две директории са едни и същи. Различават ли се ID-тата им? Да? Значи са различни.
  Не? Значи поне един файл или директория вътре се е променил.
- Всеки commit съдържа хеш на дърво и на предходен (parent) commit. Какво значи това? Познахте - ако се промени едно от тях, значи
  и commit-а ще е различен. Тоест ако променим един commit от историята - ще трябва да променим и всички след него.
- ID-то на commit еднозначно може да определи състоянието на цялото хранилище - цялата история, всички файлове и цялото съдържание
  в тях.

* Да, има вероятност за колизия, но е изключително малка - гугъл ит.

### Клони

Разгледахме какво има в `.git/objects`. В `.git` обаче има и други интересни неща:

    $ cat .git/HEAD
    ref: refs/heads/LL-1461-search-appearance

Интересно! Значи `HEAD` не е нищо повече от един файл, който съдържа името на бранч.
`LL-1461-search-appearance` е бранчът, на който съм в момента. Но `refs/heads/LL-1461-search-appearance` е път към файл:

    $ cat .git/refs/heads/LL-1461-search-appearance
    f0b841584a8a57435ec6b8a5bf5f71715907598f

Ха! Значи бранчовете са просто файлове, съдържащи ID на обект. Едва ли ще е изненада, че въпросният обект е commit:

    $ git cat-file -p f0b841584a8a57435ec6b8a5bf5f71715907598f
    tree bdc70ab34422046ed7562e7acc3e3320c46b8703
    parent 42346ebcfb9116fa78367bb6cf0e1116e856352a
    author Georgi Angelov <gangelov@asteasolutions.com> 1461151793 +0300
    committer Georgi Angelov <gangelov@asteasolutions.com> 1461151793 +0300
    gpgsig -----BEGIN PGP SIGNATURE-----
     Comment: GPGTools - https://gpgtools.org

     iQIcBAABCgAGBQJXF2g8AAoJEOQ9rPQJVVIrhikP/3yUVJeAQRcT+1aQ9M3kRdfb
     r09ED8G2I6Cuj+3fh01ZuuuaJ7GKULe+03+6XzRJvb/yGYsH2402Y1gwCyQWD8Qw
     x3Fc74JyKZ89nqOgN2KNXnjujOyisxjVtCscF5f5I6R5WJzB/qgCRpaLjRls0NUb
     Go+hyafYnQQlCnFyE3AsWKzl3BGWjbB8pwlj87B63raKyxNNCfeesGo1BTJ60Rir
     jOX8mCGfdg9mT6De1qXnNWjuAwOgriaKM3WyYCOUj7BvuaCdZHN9k7q+3CwtmQRJ
     Ub3t09RFsprr8oFRFRijCza4/+Yt35S/gT2lrnNoMc4Mr2SLXU2ouyzX4cSSL5sR
     R+BcZ9uwgvj9gY4C4l1j997vRCfBmRfvlFVkmZ3tjREz3JwLVO4xAMigYgY2EiAD
     enGeqliFmqqNTdTqW4nWpcJDQ298NflaL8Ook54cKHuvFROLoGLR+vo0tm05QrLr
     47+c75banvMdtdb3NFodAHJv9MY1FQYMa++dZ+uIvUIv9sHzCfxsBDOOdMHEjOi4
     YMY/xpfVt9UAbcGY8BYaS7lSP+7PhdEkk0pWW/0y2qCKPqXrEVVfQQWHwULVnWFF
     f7X3OH+yj0y5NFCna0RjrZujSlm8OkSeEyn0pJ08tKCrEX20XAFGgLKKeMVG6mZG
     UHaCtcNxgOG/5X9PQqF5
     =Oymd
     -----END PGP SIGNATURE-----

    [LL-1461] Fix a Facebook test

За PGP нещото ще си говорим по-натам :)

_Но наистина ли са само файлове? `git branch` трябва да прави нещо повече от това!_

Да вземем commit-а от преди малко (`c1d41c01e65da42fcf5815944cd7891240bacc5a`). Да разчовъркаме:

    $ echo "c1d41c01e65da42fcf5815944cd7891240bacc5a" > .git/refs/heads/branch-without-git-cli
    $ git branch
    [...]
    branch-without-git-cli
    [...]

Ха! Създадохме бранч без `git`. Да отидем на него без `git checkout`?

    $ echo "ref: refs/heads/branch-without-git-cli" > .git/HEAD
    $ git status
    On branch branch-without-git-cli
    Changes to be committed:
      [един милион променени неща]

Добре де. Чекаутнахме бранча, но не сме променяли файловете в работната директория. Close enough.

### Индексът

Индексът (или staging областта) е виртуалната папка, в която се съхраняват нещата, които ще се commit-ват при следващия `git commit`.
Той представлява списък от blob обекти, чиито ID-та се пазят в `.git/index`. Можем да погледнем с `git ls-files --stage`

    $ git ls-files --stage
    100644 3c2bd6360bbe0d0a2547dc56038e30383555a600 0       .dockerignore
    100644 8791029d1e95e50cb73093a2984cc883fb451b0a 0       .gitattributes
    100644 62f37653210d7e35bd97b9ed80eec1244498524b 0       .gitignore
    [...]

    $ git status
    On branch master
    Your branch is up-to-date with 'origin/master'.
    nothing to commit, working directory clean

Файловете, които виждаме в индекса не са променяни. Защо са там? Всъщност индекса пази "snapshot" на работната директория. Не е
празна папка, която се пълни само, когато се добави нещо. Това става логично като помислим, че ако изтрием нещо от работната
директория, то трябва някак да се отрази и на индекса.

### Отдалечени хранилища

Git е дистрибутирана сорс контрол система. Значи може да има повече от един "сървър", към който да си синхронизираме локалните данни.
Всяко отдалечено хранилище си има име. По подразбиране това, от което сме клонирали се нарича `origin`.

    $ git remote
    origin

Често, когато форкваме хранилище в github правим нещо от рода на:

    $ git remote add upstream <original-repo-url>
    $ git fetch upstream
    $ git merge upstream/master

Горните команди ще вземат нещата от оригиналното хранилище и ще мърджнат техния `master` в нашия.

### Remote бранчове

Сигурно знаете, че `git pull = git fetch <remote> + git merge`. Всъщност `git fetch <remote>` просто сваля всички обекти
от хранилището `<remote>` и обновява бранчовете в `.git/refs/remotes/<remote>`.

Това ни позволява да сравняваме два бранча - локален и външен. Така git знае дали има нови промени при нас, без да пита
постоянно сървъра.

Когато напишем `git checkout branch-from-remote`, ако няма `.git/refs/heads/branch-from-remote`, той просто копира
`.git/refs/remotes/origin/branch-from-remote`.

Обратно, `git push` просто прави връзка със сървъра, копира обектите, които ги няма там, и обновява отдалечения `.git/refs/heads`.

### Config

Остана един интересен файл - `.git/config`.

    $ cat .git/config
    [core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = true
    [user]
        name = Georgi Angelov
        email = gangelov@asteasolutions.com
        signingkey = 0955522B
    [remote "origin"]
        url = git@github.com:asteasolutions/livelist-application-server.git
        fetch = +refs/heads/*:refs/remotes/origin/*
    [branch "master"]
        remote = origin
        merge = refs/heads/master
    [branch "719-web-project"]
        remote = origin
        merge = refs/heads/719-web-project
    [...]

Тук се съхраняват настройките за конкретното хранилище. Виждаме, например, че съм му казал да използва `gangelov@asteasolutions.com`
вместо личния ми имейл, когато commit-вам по LiveList. Има още доста интересни опции, които може да видите с `man git-config`.

Виждаме също, че има връзка към отдалечено хранилище с име `origin` и адрес `git@github.com:asteasolutions/livelist-application-server.git`.

Освен това, бранчът `master` е настроен да следи `master` от `origin`. Затова, когато напиша `git pull`, то се сеща да обнови и него.

### Бонус

Stash-натите неща са commit-и:

    $ cat .git/refs/stash
    ccd6960cf50f354850daed5d54d5034465db71ff

    $ git cat-file -p ccd6960cf50f354850daed5d54d5034465db71ff
    tree a0be53019ff34e304c1f2eb18c552ac3bee74a98
    parent 9b7af954a871168e8904b39ff050cae76a054c56
    parent 3e52596be9de23e01a4ba10636651b486a81c365
    parent 1d70c3bcb26461711e1d371639c3a6bc4a413404
    author Georgi Angelov <gangelov@asteasolutions.com> 1461185740 +0300
    committer Georgi Angelov <gangelov@asteasolutions.com> 1461185740 +0300

    WIP on master: 9b7af95 Merge pull request #542 from asteasolutions/internal-tasks

## Заключение

Следващият път като се омаже нещо с git, помислете какво се е случило отдолу. Най-вероятно доста лесно ще стигнете до решението, представяйки си връзките между нещата под капака.
