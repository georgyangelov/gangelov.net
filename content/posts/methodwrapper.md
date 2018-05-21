---
title: "MethodWrapper - една история за калпави библиотеки, singleton-и, scope gate-ове и метапрограмиране"
date: 2016-09-29T21:20:25+03:00
draft: true
description: Човек и добре да живее, рано или късно му се налага да използва недомислена библиотека. Добре де, ако пише на Node.js му се налага постоянно... Просто ние, рубистите, не сме свикнали. (Споменах ли, че ползваме макове и сме готини?)
---

Човек и добре да живее, рано или късно му се налага да използва недомислена библиотека. Добре де,
ако пише на Node.js му се налага постоянно... Просто ние, рубистите, не сме свикнали. (Споменах ли,
че ползваме макове и сме готини?)

И какво правим, когато библиотеката ни е необходима, но един детайл от нея не е направен с мисълта за
нашия (по-общия) случай?

Въпросната библиотека е `hubspot-ruby`, но тя е по-скоро детайл и не е целта на този пост.
Достатъчно ви е да знаете, че Hubspot е "някакво нещо" за analytics, на което се пращат данни. Външен сървис.

## Проблемът

Та, `hubspot-ruby` се използва така:

```ruby
Hubspot.configure hapikey: "YOUR_API_KEY"

Hubspot::Contact.create! "email@address.com", firstname: "First", lastname: "Last"
```

И тук лежи проблемът. Видяхте ли го? Добре де, примерът е най-честия use-case и е нормално да
почнете да го забелязвате чак като се сблъскате с него. Аз го наричам "проблемът singleton".

На даден етап трябваше да направя един проект multi-tenant. Тоест, сървър с два клиента и привидно разделени данни в базата.

Част от това е изискването в зависимост от клиента да се пращат данни към различен Hubspot акаунт.
Това значи различен `API_KEY` в зависимост от request параметрите.

Сега видяхте ли проблема?

Ето кода на `Hubspot::Config`:

```ruby
require 'logger'

module Hubspot
  class Config

    CONFIG_KEYS = [:hapikey, :base_url, :portal_id, :logger]
    DEFAULT_LOGGER = Logger.new('/dev/null')

    class << self
      attr_accessor *CONFIG_KEYS

      def configure(config)
        config.stringify_keys!
        @hapikey = config["hapikey"]
        @base_url = config["base_url"] || "https://api.hubapi.com"
        @portal_id = config["portal_id"]
        @logger = config['logger'] || DEFAULT_LOGGER
        self
      end

      def reset!
        @hapikey = nil
        @base_url = "https://api.hubapi.com"
        @portal_id = nil
        @logger = DEFAULT_LOGGER
      end

      [...]
    end

    reset!
  end
end
```

Ако все още не сте разбрали какъв е проблемът - `goto Проблемът`.

---

Конфигурацията е глобална. Пази се в (нещо като) статични променливи. Не можем да имаме по едно и също
време две извиквания на API-то, използващи различни ключове. Защото ключът се взима от `Hubspot::Config.hapikey`.

Ако не сте сигурни защо това е проблем - нишки и дублиране на код. Но основно нишки ... и дублиране на код.

## Може би има лесно решение?

Когато осъзнах това, минах по стандартните стъпки:

- Има ли нова версия на библиотеката, в която това да е оправено?
- Има ли pull request за това?
- Има ли друга библиотека, която върши това по-добре?
- Има ли лесен начин да се monkey-patch-не?
- Лесно ли ще е да го оправя в библиотеката?

За съжаление, отговорът на всички тези въпроси е твърдо _нье_.

## Може би решение?

В този момент става ясно, че ще трябва да се оправят нещата в нашия код.
В такива ситуации обикновено започвам да пиша код, представяйки си, че имам някакви методи и да
рефакторирам докато не се получи нещо приемливо.

```ruby
module HubspotService
  def create_hubspot_user(app_user, ...)
    Hubspot.configure(hubspot_api_key)

    [...]
  ensure
    Hubspot.reset!
  end

  def update_hubspot_user(app_user, ...)
    Hubspot.configure(hubspot_api_key)

    [...]
  ensure
    Hubspot.reset!
  end

  def submit_hubspot_form(form)
    Hubspot.configure(hubspot_api_key)

    [...]
  ensure
    Hubspot.reset!
  end

  private

  def hubspot_api_key
    [...]
  end
end
```

Какво не ни харесва тук? Повторението и това, че е супер лесно да се забрави някое от извикванията.
`Hubspot.reset!` реално не е задължително, но така се подсигуряваме, че ако някъде изпуснем `configure`,
то API call-ът ще гръмне, вместо да прати данни към грешния акаунт. Това води до трудни за намиране бъгове.

## Wishful thinking

Окей. `configure` и `reset!` трябва да вървят в комплект. Ами да ги направим на един метод.

```ruby
module HubspotService
  def create_hubspot_user(app_user, ...)
    with_api_key do
      [...]
    end
  end

  def submit_hubspot_form(form)
    with_api_key do
      [...]
    end
  end

  private

  def with_api_key
    Hubspot.configure(hubspot_api_key)

    yield
  ensure
    Hubspot.reset!
  end

  [...]
end
```

Така е доста по-добре. Така няма начин да забравим едно от нещата. Ако забравим да викнем `with_api_key`,
Hubspot API-то ще хвърли грешка и веднага ще го забележим.

Да решим и проблемът с повторението. Не искаме да викаме `with_api_key` постоянно.

В този момент вече бях в пълен _wishful thinking_ режим. Нямаше ли да е яко да можем да направим така:

```ruby
module HubspotService
  with_api_key do
    def create_hubspot_user(app_user, ...)
      [...]
    end

    def submit_hubspot_form(form)
      [...]
    end
  end

  def self.with_api_key
    [... wat do here ...]
  end

  [...]
end
```

Почнах вече да мисля как да имплементирам `with_api_key`. И тук обикновено сме поставени пред избор -
"Да генерализирам или да не генерализирам?". Тоест дали да е `with_api_key` или
`wrap_methods with: :configure_hubspot`

Много често по-генералното решение е по-чисто, по-ясно и се пише по-лесно.

Отново с помощта на въображението ни, ще си представим, че имаме миксин `MethodWrapper`, който работи така:

```ruby
module HubspotService
  extend MethodWrapper

  wrap_methods with: :configure_hubspot do
    def create_hubspot_user(app_user, ...)
      [...]
    end

    def submit_hubspot_form(form)
      [...]
    end
  end

  def configure_hubspot
    Hubspot.configure(hubspot_api_key)

    yield
  ensure
    Hubspot.reset!
  end

  [...]
end
```

Всъщност, за да решим и проблема с нишките и race-condition-ите, който има този код,
трябва да променим малко `configure_hubspot`:

```ruby
HUBSPOT_LOCK = Mutex.new

def configure_hubspot
  HUBSPOT_LOCK.synchronize do
    begin
      Hubspot.configure(hubspot_api_key)

      yield
    ensure
      Hubspot.reset!
    end
  end
end
```

Така остава единствено да имплементираме `MethodWrapper`. И тук става интересно.

## MethodWrapper - примерът

Ето как искаме да се използва `MethodWrapper`:

```ruby
class Test
  extend MethodWrapper

  wrap_methods with: :logger do
    def test_one(arg)
      puts "This should be wrapped - test_one(#{arg})"
    end
  end

  def test_two
    puts 'This should not be wrapped'
  end

  def logger
    puts "Called method"
    yield
    puts "End method"
  end
end

Test.new.test_one 'arg'
#=> Called method
#=> This should be wrapped - test_one('arg')
#=> End method

Test.new.test_two
#=> This should not be wrapped
```

Това спокойно може да се напише на тест. <b>Ето, правим TDD.</b>

## MethodWrapper - имплементацията

Отваряме един празен файл с модул:

```ruby
module MethodWrapper
  # TODO: Code
end
```

Тъй като горе сме ползвали `extend` вместо `include`, тук ще пишем "нормални" методи,
без `self.` или `class << self`:

```ruby
module MethodWrapper
  def wrap_methods(with:)
    # TODO: Code
  end
end
```

### module_eval

Първият трик е как да накараме методите, които са дефинирани в блока да се дефинират в оригиналния модул:

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    self.module_eval(&block)
  end
end
```

_В този пример и само с `yield` ще работи, но това ще ни трябва по-натам така или иначе._

_И да, знам, че `self.` е излишно, но прави кода по-ясен за примера :)_

Всяко едно парче Ruby код си има контекст. Част от този контекст е `self` - имплицитната променлива "текуща инстанция".
Освен `self` има и нещо, наречено "текущ клас", което не може да се достъпи директно като променлива (няма `self_class`),
но влияе на това как работи `def`. Когато се изпълни `def method_name`, то `method_name` се дефинира в текущия клас.

`module_eval` (и еквивалентът му за класове `class_eval`) изпълнява подадения му блок с променен текущ клас.
Ето прост пример:

```ruby
class Test
end

Test.module_eval do
  def test
    puts 'baba'
  end
end

Test.new.test #=> baba
```

Но да се върнем на `wrap_methods`. Все още не правим нищо, но поне вече и не чупим нищо.

### You got me hooked

Вече дефинираме методите, подадени на блока. Сега трябва да разберем кои методи са дефинирани вътре в блока,
за да ги "модифицираме".

В Ruby има специални класови методи, които са hook-ове. Например, метода `method_added` се извиква
при всяко дефиниране на метод.

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    self.module_eval do
      def self.method_added(method_name)
        # TODO: Code
        puts "#{method_name} defined"
      end
    end

    self.module_eval(&block)
  end
end
```

Горното ще изпише `<името на метода> defined` за всеки метод дефиниран след като дефинираме `method_added`.
Включително и за методи дефинирани извън блока, но след извикването на `wrap_methods`.

Можем да решим този проблем с един флаг:

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    wrapping = true

    self.module_eval do
      def self.method_added(method_name)
        if wrapping
          # TODO: Code
          puts "#{method_name} defined"
        end
      end
    end

    self.module_eval(&block)

    wrapping = false
  end
end
```

### Scope gate-ове

За съжаление, последното парче код няма да работи. Грешката ще е нещо от рода на (по спомен)
`Undefined variable or method "wrapping"`.

В Ruby някои синтактични конструкции представляват "scope gate"-ове. Тоест, бариери, които
разделят скоупове. В един скоуп се съдържат локалните променливи, дефинирани в него. Когато дефинираме метод с `def`, в тялото на метода нямаме достъп до локални променливи, дефинирани извън него.

Прост пример:

```ruby
wat = 'wow'

def test
  wat
end

test #=> NoMethodError: Undefined variable or method `wat`
```

Това важи за ключовите думи `def`, `class` и `method`. Не е част от семантиката на дефиниране на метод.
Тоест, ако не дефинираме метода с `def`, всичко ще е наред.

```ruby
wat = 'wow'

define_method :test do
  wat
end

test #=> 'wow'
```

Ако се върнем на `MethodWrapper` - там можем да използваме `define_singleton_method` или `singleton_class.define_method`
(заради `self.`-а в дефиницията - това е класов метод):

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    wrapping = true

    self.module_eval do
      define_singleton_method :method_added do |method_name|
        # TODO: Code
        puts "#{method_name} defined" if wrapping
      end
    end

    self.module_eval(&block)

    wrapping = false
  end
end
```

### Игра на методи

Вече можем да разберем кои методи са дефинирани вътре в блока, подаден на `wrap_methods`.
Следващото, което трябва да направим, е да го сменим с наша версия, около която се вика подаденият метод.

Искаме от това:

```ruby
wrap_methods with: :pretty do
  def doge
    puts 'much meta'
  end
end

def pretty
  puts 'such code'
  yield
  puts 'wow'
end
```

да стане това:

```ruby
def doge
  pretty do
    puts 'much meta'
  end
end

doge
#=> such code
#=> much meta
#=> wow
```

Това можем да го направим по (поне) два начина, но по-простият сякаш е да го направим на стъпки:

1. Копираме метода `doge` и му даваме ново име `unwrapped_doge`
2. Правим нов метод `doge`, който съдържа `pretty { unwrapped_doge }`

```ruby
alias_method :unwrapped_doge, :doge

def doge(*args)
  pretty do
    unwrapped_doge(*args)
  end
end
```

`alias_method` копира метод, а `*args` приема и предава всички подадени параметри.

### Вторият най-известен ексепшън

Вече знаем какво трябва да направим с методите - трябва само да го прехвърлим в `MethodWrapper`:

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    wrapper_method = with # `with` е хубаво име за параметър, но ужасно име за локална променлива
    wrapping = true

    self.module_eval do
      define_singleton_method :method_added do |method_name|
        if wrapping
          # alias_method :unwrapped_doge, :doge
          alias_method "unwrapped_#{method_name}", method_name

          # def doge
          define_method method_name do |*args|
            # pretty do
            self.send wrapper_method do
              # unwrapped_doge(*args)
              self.send "unwrapped_#{method_name}", *args
            end
          end
        end
      end
    end

    self.module_eval(&block)

    wrapping = false
  end
end
```

Това работи ... почти. Ако решите да го пуснете, ще ви поздрави с вторият най-известен ексепшън
в програмирането - `StackOverflow` (ако се чудите, първият е `NullPointerException`).

Сетихте ли се защо ще гръмне това? Ами да - дефинираме метод в hook-а за дефиниране на метод.

Това ще го решим лесно - трябва да разделим слушането за методи от модификацията им:

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    wrapper_method = with
    wrapping = true
    methods_to_wrap = []

    self.module_eval do
      define_singleton_method :method_added do |method_name|
        methods_to_wrap << method_name if wrapping
      end
    end

    self.module_eval(&block)

    wrapping = false

    methods_to_wrap.each do |method_name|
      alias_method "unwrapped_#{method_name}", method_name

      define_method method_name do |*args|
        self.send wrapper_method do
          self.send "unwrapped_#{method_name}", *args
        end
      end
    end
  end
end
```

Супер - просто разместихме едно парче код и дефинирахме един масив.

И вече работи! Хора, направихме го! Готово. Край. The end. Nothing more to see here.

### Бонус нивото - Primum non nocere

Всичко е дъги и еднорози, но оставяме боклук след себе си. Методът `self.method_added`, който дефинираме
остава. И по-лошо - предефинира съществуващия такъв (ако има).

Затова, нека се изолираме от self. Ще си направим собствен (анонимен) модул, в който да правим магиите:

```ruby
module MethodWrapper
  def wrap_methods(with:, &block)
    wrapper_method = with
    wrapping = true
    methods_to_wrap = []

    # Ето този ред се промени
    wrapper_module = Module.new do
      define_singleton_method :method_added do |method_name|
        methods_to_wrap << method_name if wrapping
      end
    end

    # Това от `self` стана на `wrapper_module`
    wrapper_module.module_eval(&block)

    wrapping = false

    # Този код го оградихме в `module_eval`
    wrapper_module.module_eval do
      methods_to_wrap.each do |method_name|
        alias_method "unwrapped_#{method_name}", method_name

        define_method method_name do |*args|
          self.send wrapper_method do
            self.send "unwrapped_#{method_name}", *args
          end
        end
      end
    end

    # Добавихме и този ред
    self.send :include, wrapper_module
  end
end
```

Тук променихме четири неща:

1. Създаваме нов (анонимен) модул с `Module.new`. Дефинираме `method_added` в него, вместо в `self`.
2. Дефинираме методите от блока в нашия модул, не в `self`.
3. Променяме методите в нашия модул, не в `self`.
4. "Инжектираме" нашия модул в `self`. Това става с `include wrapper_module`, но понеже `include` е private метод, трябва да ползваме `send`.

Така бърникаме само в нашия модул и накрая го "закачаме" за оригиналния. Не пречим на външен код, който разчита на `method_added`.

## Заключение

Метапрограмирането е като водата - в малки количества е много полезно. Но внимавайте да не нагълтате прекалено
много.

За щастие - границата е много размита (pun not really intended).
В повечето случаи бих предпочел повече сложност на едно място, вместо малки количества сложност на много места.
Пак като водата - искате да е в чашата, не разлята на пода.
Разбира се, един варел с вода на масата също не е полезен.

Заслужаваше ли си усилието `MethodWrapper`? Може би не. Може би бих си wrap-нал кода на всеки от методите. А може би не.
Зависи на колко места се използва и какво спестява.

В крайна сметка, това е едно доста добро упражнение на тема "Разбирам ли наистина Ruby?" и (поне на мен) ми беше интересно.
