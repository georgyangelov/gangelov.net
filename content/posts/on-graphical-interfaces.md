---
title: "За графичните интерфейси"
date: 2018-05-21T21:20:25+03:00
draft: false
description: Традиционно, когато мислим за графични интерфейси, си мислим за обекти, които описват всеки един елемент на екрана. Това не е единствената възможност.
---

Традиционно, когато мислим за графични интерфейси, си мислим за обекти, които описват всеки един елемент на екрана.

Искаме розов правоъгълник? Няма проблем:

```js
var rect = new Rectangle({width: 100, height: 100});
rect.color = Colors.Pink;
screen.addChild(rect);
```

Правоъгълникът трябва да бъде в центъра на екрана? Отново, няма проблем:

```js
rect.x = screen.width / 2 - rect.width / 2;
rect.y = screen.height / 2 - rect.height / 2;
```

Какво се случва, обаче, когато прозорецът на приложението си промени размерите? Очевидно искаме да преизчислим координатите на базата на новите размери:

```js
screen.on('resize', () => {
  rect.x = screen.width / 2 - rect.width / 2;
  rect.y = screen.height / 2 - rect.height / 2;
});
```

ОК, това не изглежда твърде добре. Защо? Защото променяме състояние, а както знаем, промяната на състояние по подобен начин води до експлозия от възможни резултати. Всеки един от тези възможни резултати трябва да бъде предвиден от програмиста. Това рано или късно води до лудост.

Да погледнем пак правоъгълника. Всяка една негова характеристика е поле, което се съдържа в този обект. Включително не-собствени характеристики като тези за позициониране. Защо трябва позицията на екрана да бъде част от знанието на правоъгълника? По-скоро не трябва да бъде. Но тогава къде да го сложим?

Един валиден отговор би бил - в друг UI обект, който не рисува на екрана, а позиционира елементите в себе си.

```js
var rect = new Rectangle({width: 100, height: 100});
rect.color = Colors.Pink;

var centered = new CenteredLayout();
centered.child = rect;

screen.addChild(centered);
```

Сега не трябва ръчно да позиционираме всеки един UI елемент, можем да преизползваме знанието как се центрират неща. Но вече имаме два вида обекти - обекти, които рисуват, и обекти, които позиционират. А сметките, които правехме преди все още са тук, просто са скрити в този нов "позициониращ" клас:

```ts
class CenteredLayout {
  child: View;

  addedAsChild(parent) {
    parent.on('resize', updateChildPosition);
    updateChildPosition();

    parent.addChild(child);
  }

  updateChildPosition() {
    child.x = parent.width / 2 - child.width / 2;
    child.y = parent.height / 2 - child.height / 2;
  }
}
```

Все още правим същото нещо - слушаме за промени в бащата, след което ъпдейтваме децата. Все още променяме състояние.

ОК, премълчах част от истината преди малко. Въпреки, че почти винаги е по-добре да не променяме състояние ако можем, често нямаме такава възможност. В крайна сметка UI-а *трябва* да се променя, без значение колко ни е страх нас от промяна. Честно казано, промяната на състояние не е проблемът - проблемът е промяната на много състояния на и от различни места.

Какво ще стане ако вместо отделно състояние за всеки UI обект, имаме само един обект със състояние, който може да бъде променян?

```js
var state = {
  rect: {
    width: 100,
    height: 100,
    color: Colors.Pink,
    x: 0,
    y: 0
  }
};

function center(element, parent) {
  element.x = parent.width / 2 - element.width / 2;
  element.y = parent.height / 2 - element.height / 2;
}

screen.on('resize', () => {
  center(state.rect, screen);
});

center(state.rect, screen);
```

Сега ни трябва начин да пренесем това състояние върху реалния интерфейс:

```js
var rect = new Rectangle();

function applyState() {
  rect.width = state.rect.width;
  rect.height = state.rect.height;
  rect.color = state.rect.color;
  rect.x = state.rect.x;
  rect.y = state.rect.y;
}
```

Сега, когато променим нещо в състоянието, просто извикваме `applyState()` и интерфейсът ще бъде обновен.

Но това е доста код и изглежда грозен и неестествен. Това е защото наистина *e* неестествен. Неестествен е за системата, в която всеки един UI елемент е обект, който държи собствено състояние и трябва да бъде модифициран, за да се случат промени на екрана. Това е така нареченият retained-mode подход.

Да започнем отначало. Да си представим, че вместо всеки един UI елемент да беше обект, имахме само примитиви за рисуване и метод, който се извиква на всеки кадър, 60 пъти в секунда:

```js
function drawApplication() {
  var x = screen.width / 2 - 50;
  var y = screen.height / 2 - 50;

  screen.drawRect({x: x, y: y, width: 100, height: 100, color: Colors.Pink});

  // More draw calls here for other UI elements
}
```

Този път няма нужда да правим нищо, за да обновяваме позицията на правоъгълника. Получаваме това наготово, защото координатите по естествен път се преизчисляват на всеки кадър. Това се нарича immediate-mode UI.

Но какво става ако искаме да нарисуваме нещо по-сложно? Нещо, което не е директно извикване на API-то за рисуване. Лесно, ще го пакетираме във функция:

```js
function drawRectWithUnicorns(parent, x, y, width, height, color) {
  // Code omitted as unicorns are shy...
}

function drawApplication() {
  var x = screen.width / 2 - 50;
  var y = screen.height / 2 - 50;

  drawRectWithUnicorns(screen, x, y, 100, 100, color);

  // More draw calls here for other UI elements
}
```

Това е нещото, което някои гейм разработчици правят през цялото време - използвайки чист екран на всеки фрейм, рисувайки на него и нямайки нужда да управляват много експлицитни състояния. Когато се налага да има някакво място за съхранение на състояние - винаги можем да създадем този `state` обект, както направихме преди малко.

За съжаление, този подход също бързо става неуправляем - стигаме до функция, която създава целия UI на приложението, извиквайки един тон методи за рисуване с много параметри. Също, все още смятаме позицията ръчно.

Да направим няколко подобрения. Първо, малко подготовка. Да наречем тези рисуващи функции `render` и да ги опаковаме в обекти.

```ts
class RectWithUnicorns {
  constructor(props) {
    this.props = props;
  }

  render(parent) {
    // Code here uses this.props
  }
}

class Application {
  render(screen) {
    var rect = new RectWithUnicorns({
      width: 100,
      height: 100,
      color: Color.Pink,
      x: screen.width / 2 - 50,
      y: screen.height / 2 - 50
    });

    rect.render(screen);

    // More render calls for other UI elements
  }
}
```

Не забравяйте, че `render` функциите все още се извикват на всеки кадър, просто ги обвихме в обекти. Защо? Защото обектите са много добри в скриването на детайли и опаковането на данни.

Сега можем да върнем абстракцията за центриране от преди малко. Можем да го направим както с функция:

```js
function center(element, parent) {
  element.x = parent.width / 2 - element.width / 2;
  element.y = parent.width / 2 - element.height / 2;
}

class Application {
  render(screen) {
    var rect = new RectWithUnicorns({
      width: 100,
      height: 100,
      color: Color.Pink
    });

    center(rect, screen);

    rect.render(screen);

    // More render calls for other UI elements
  }
}
```

Така и с "позициониращ" обект, както направихме в началото:

```ts
class Centered {
  constructor(element) {
    this.element = element;
  }

  render(parent) {
    this.element.x = parent.width / 2 - this.element.width / 2;
    this.element.y = parent.height / 2 - this.element.height / 2;

    this.element.render(parent);
  }
}

class Application {
  render(screen) {
    var rect = new RectWithUnicorns({
      width: 100,
      height: 100,
      color: Color.Pink
    });

    var centeredRect = new Centered(rect);

    centeredRect.render(screen);

    // More render calls for other UI elements
  }
}
```

Това, което написахме тук обновява позицията правилно когато прозорецът се преоразмери, защото `render` методът се извиква на всеки кадър. И това обновяване се случва естествено, защото нямаше нужда да се замисляме дали нещо трябва да бъде ъпдейтнато при някое събитие или не. То просто се случва. Синхронизирането на данните с UI-а вече е просто - правим една (`render`) функция, която взима данни и ги показва (рисува) на екрана.

Можем да направим кода и малко по-приятен за окото:

```ts
class Application {
  render(screen) {
    new Centered(
      new RectWithUnicorns({
        width: 100,
        height: 100,
        color: Color.Pink
      })
    ).render(screen);

    // More render calls for other UI elements
  }
}
```

Сега добавянето на нови UI елементи е тривиално и добре композируемо:

```ts
class Application {
  render(screen) {
    new Rows([
      new RectWithUnicorns({ /* ... */ }),

      new Columns([
        new Button({label: 'Previous', /* ... */}),
        new Label({label: 'Unicorn 1'}),
        new Button({label: 'Next', /* ... */}),
      ]);
    ]).render(screen);
  }
}
```

Какво става ако ни трябва състояние? Можем да вкараме `state` обекта от преди малко и да държим данните там:

```ts
class Application {
  UNICORNS = [ /* ... */ ];

  constructor() {
    this.state = {
      unicornIndex: 0
    };
  }

  render(screen) {
    new Rows([
      new RectWithUnicorns({unicorn: this.UNICORNS[this.state.unicornIndex]}),

      new Columns([
        new Button({label: 'Previous', onClick: () => this.state.unicornIndex -= 1}),
        new Label({label: 'Unicorn ' + this.state.unicornIndex}),
        new Button({label: 'Next', onClick: () => this.state.unicornIndex += 1}),
      ]);
    ]).render(screen);
  }
}
```

Отново, когато кликнем някой от бутоните, няма нужда да се грижим ръчно да ъпдейтваме (1) `RectWithUnicorns`, (2) label-а, или (3) да включваме/изключваме "Previous" и "Next". Няма нужда да се замисляме за тези неща. Това е страхотно!

Правейки още една стъпка, представете си, че имахме XML-подобен синтаксис за горния код и вместо да викаме `.render(screen)` накрая, можем просто да върнем обекта, който трябва да бъде нарисуван:

```jsx
class Application {
  // ...

  render() {
    return (
      <Rows>
        <RectWithUnicorns unicorn={this.UNICORNS[this.state.unicornIndex]} />

        <Columns>
          <Button onClick={() => this.state.unicornIndex -= 1}>Previous</Button>
          <Label>Unicorn {this.state.unicornIndex}</Label>
          <Button onClick={() => this.state.unicornIndex += 1}>Next</Button>
        </Columns>
      </Rows>
    );
  }
}
```

Остана само едно нещо. Представете си, че имаме retained-mode система и искаме да използваме immediate-mode подходът по този начин. Самата система изисква използването на обекти, които се запазват между кадрите. Как да трансформираме това...:

```ts
render(screen) {
  return new Rectangle({
    x: screen.width / 2 - 50,
    y: screen.height / 2 - 50,
    width: 100,
    height: 100
  });
}
```

...в това:

```ts
var rect = new Rectangle({width: 100, height: 100, ...});
screen.on('resize', () => {
  rect.x = screen.width / 2 - 50;
  rect.y = screen.height / 2 - 50;
});

screen.addChild(rect);
```

Един начин би бил да изтриваме всички retained-mode обекти и да ги пресъздаваме на всеки кадър. Но това би било много "скъпо", защото retained-mode обектите са тежки - те съдържат много неща и се подреждат в йерархии.

Друг начин би бил да имаме някаква логика, която автоматично сравнява характеристиките на тези обекти с характеристиките, които искаме, и обновява само тези, за които има нужда. Това включва промяна на полета, създаване на нови retained-mode обекти и изтриване на стари такива.

Сега, HTML е retained-mode система. Тази логика по ъпдейтване е на практика това, което наричат Virtual DOM - един адаптер между immediate-mode подход и retained-mode система.

Хей, вижте, (почти) преоткрихме React! :)
