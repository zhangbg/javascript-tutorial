# Улучшаем сжатие кода

Здесь мы обсудим разные приёмы, которые используются, чтобы улучшить сжатие кода.

[cut]

## Больше локальных переменных

Например, код jQuery обёрнут в  функцию, запускаемую "на месте".

```js
(function(window, undefined) {
  // ...
  var jQuery = ...

  window.jQuery = jQuery; // сделать переменную глобальной

})(window);
```

Переменные `window` и `undefined` стали локальными. Это позволит сжимателю заменить их на более короткие.

## ООП без прототипов

Приватные переменные будут сжаты и заинлайнены.

Например, этот код хорошо сожмётся:

```js
function User(firstName, lastName) {
  var fullName = firstName + ' ' + lastName;

  this.sayHi = function() {
    showMessage(fullName);
  }

  function showMessage(msg) {
    alert( '**' + msg + '**' );
  }
}
```

..А этот -- плохо:

```js
function User(firstName, lastName) {
  this._firstName = firstName;
  this._lastName = lastName;
}

User.prototype.sayHi = function() {
  this._showMessage(this._fullName);
}

User.prototype._showMessage = function(msg) {
  alert( '**' + msg + '**' );
}
```

Сжимаются только локальные переменные, свойства объектов не сжимаются, поэтому эффект от сжатия для второго кода будет совсем небольшим.

При этом, конечно, нужно иметь в виду общий стиль ООП проекта, достоинства и недостатки такого подхода.

## Сжатие под платформу, define

Можно делать разные сборки в зависимости от платформы (мобильная/десктоп) и браузера.

Ведь не секрет, что ряд функций могут быть реализованы по разному, в зависимости от того, поддерживает ли среда выполнения нужную возможность.

### Способ 1: локальная переменная

Проще всего это сделать локальной переменной в модуле:

```js
(function($) {

*!*
  /** @const */
  var platform = 'IE';
*/!*

  // .....

  if (platform == 'IE') {
    alert( 'IE' );
  } else {
    alert( 'NON-IE' );
  }

})(jQuery);
```

Нужное значение директивы можно вставить при подготовке JavaScript к сжатию.

Сжиматель заинлайнит её и оптимизирует соответствующие IE.

### Способ 2: define

UglifyJS и GCC позволяют задать значение глобальной переменной из командной строки.

В GCC эта возможность доступна лишь в "продвинутом режиме" работы оптимизатора, который мы рассмотрим далее (он редко используется).

Удобнее в этом плане устроен UglifyJS. В нём можно указать флаг `-d SYMBOL[=VALUE]`, который заменит все переменные `SYMBOL` на указанное значение `VALUE`. Если `VALUE` не указано, то оно считается равным `true`.

Флаг не работает, если переменная определена явно.

Например, рассмотрим код:

```js
// my.js
if (isIE) {
  alert( "Привет от IE" );
} else {
  alert( "Не IE :)" );
}
```

Сжатие вызовом `uglifyjs -d isIE my.js` даст:

```js
alert( "Привет от IE" );
```

..Ну а чтобы код работал в обычном окружении, нужно определить в нём значение переменной по умолчанию. Это обычно делается в каком-то другом файле (на весь проект), так как если объявить `var isIE` в этом, то флаг `-d isIE` не сработает.

Но можно и "хакнуть" сжиматель, объявив переменную так:

```js
// объявит переменную при отсутствии сжатия
// при сжатии не повредит
window.isIE = window.isIE || getBrowserVersion();
```

## Убираем вызовы console.*

Минификатор имеет в своём распоряжении дерево кода и может удалить ненужные вызовы.

Для UglifyJS это делают опции компилятора:

- `drop_debugger` -- убирает вызовы `debugger`.
- `drop_console` -- убирает вызовы `console.*`.

Можно написать и дополнительную функцию преобразования, которая убирает другие вызовы, например для `log.*`:

```js
var uglify = require('uglify-js');
var pro = uglify.uglify;

function ast_squeeze_console(ast) {
  var w = pro.ast_walker(),
    walk = w.walk,
    scope;
  return w.with_walkers({
    "stat": function(stmt) {
      if (stmt[0] === "call" && stmt[1][0] == "dot" && stmt[1][1] instanceof Array && stmt[1][1][0] == 'name' && stmt[1][1][1] == "log") {
        return ["block"];
      }
      return ["stat", walk(stmt)];
    },
    "call": function(expr, args) {
      if (expr[0] == "dot" && expr[1] instanceof Array && expr[1][0] == 'name' && expr[1][1] == "console") {
        return ["atom", "0"];
      }
    }
  }, function() {
    return walk(ast);
  });
};
```

Эту функцию следует вызвать на результате `parse`, и она пройдётся по дереву и удалит все вызовы `log.*`.
