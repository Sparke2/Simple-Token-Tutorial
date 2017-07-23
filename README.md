# Пишем смарт-контракт цифрового актива в блокчейне Ethereum на Solidity

  В этом примере мы рассмотрим как написать на Solidity контракт простого токена. Токен - это цифровой актив на блокчейне Ethereum. Его можно хранить на своем кошельке и передавать другим пользователям. У токена могут быть и другие специфичные для него функции в зависимости от того какую цель мы преследуем, создавая данный цифровой актив.

  Там мы, к примеру, можем написать токен для голосования. Обладатели такого токена смогут голосовать в специально написанном для этого контракте и голоса пользователей, у которых на балансе больше токенов, будут иметь больший вес и наоборот.

## Что нам нужно знать о смарт-контрактах на Solidity?

### 1. Синтаксис.
  Его вы можете изучить [по этим урокам](https://ethereumbuilders.gitbooks.io/guide/content/en/solidity_tutorials.html). Так же можно ознакомиться с [документацией по Solidity](http://solidity.readthedocs.io/en/develop/solidity-by-example.html).

### 2. Базовые представления о том как работает контракт.
  Смарт-контракты в блокчейне исполняются только когда к ним непосредственно обращаются транзакцией. Сами собой с течением времени смарт-контракты не исполняются. Как следствие задачи реального времени контракты решать не могут, если контракт должен исполниться в какое-то определенное время, значит в это время к нему должен кто-либо обратиться.

  Т.к. контракты работают только когда к ним обращаются транзакцией, то в ходе исполнения контракта всегда доступна информация о транзакции, которой к нему обратились.
  Информация о транзакции хранится в специальной структуре `msg` и содержит следующие переменные:

1. `msg.sender` - адрес отправителя транзакции (тот кто вызывает контракт)
2. `msg.value`  - количество Эфира, которое передаются в транзакции
3. `msg.data`   - кодировка функции вызова с её параметрами (в нашем случае не потребуется)

  Так же доступна информация о блоке, в который попал вызов данной транзакции в специальной структуре `block`:

1. `block.number` - номер блока
2. `block.timestamp` - время, в которое был добыт блок в UNIX секундах

Подробнее со всеми переменными вы сможете ознакомиться в документации по Solidity.

### 3. Газ.
  Все вычисления в ходе исполнения контракта оплачиваются **газом**. Газ - это Эфир, который в виде комиссии уплачивается в момент произведения транзакции. Ресурсоемкие вычисления требуют больше газа, а соответственно и стоят дороже. У каждого блока есть **газлимит** - максимальное количество газа, которое может быть использовано суммарно всеми транзакциями, входящими в этот блок. Обычно **газлимит** блока составляет 4,700,000 в настоящее время.

  Это означает что если мы пытаемся послать 3 транзакции, каждая из которых расходует 2,000,000 газа, то первые две транзакции войдут в блок, а третья транзакция будет ждать следующего блока чтобы быть обработанной.
  
  Данная механика введена для предотвращения DDOS-атак на сеть, как в случае если бы в один блок попали множественные ресурсоемкие транзакции, заставляющие майнера произвести огромное количество различных вычислений для того чтобы сформировать блок.
  
### 4. Специфика вычислений.
  1) Избегать массивов и циклов. Вы не сможете перебрать массив из 100 000 элементов циклом, т.к. операция инкремента (увеличение переменной-счетчика на единицу на каждой итерации), выполненная 100 000 раз, расходует больше 4,700,000 газа и соответственно такая транзакция не может быть включена ни в один блок из-за газлимита.
  2) Использовать хеш-таблицы (mapping). Наиболее часто используемым хранилищем переменных в Solidity является структура `mapping (type1 => type2) mappingName;`. Хеш-таблица представляет из себя список связей ключ->значение.
  3) Виртуальная машина Ethereum не поддерживает числа с плавающей точкой и отрицательные числа. Числовые значения как правило хранятся в переменных типа `uint`.
  4) За все вычисления всегда платит вызывающий транзакцию пользователь. Контракт не может сам оплачивать вычисления. Даже если в ходе выполнения транзакции один контракт должен вызвать другой контракт, пользователь все равно оплачивает все необходимые действия (вызов как первого контракта, так и второго) своим Эфиром. Если в ходе выполнения транзакции газ закончился и его не хватило на то чтобы полностью исполнить транзакцию, все изменения откатываются, газ расходуется, транзакция завершается неудачно с ошибкой `Out Of Gas`.
  5) Все вызовы контракта происходят за одну транзакцию.  Даже если в ходе выполнения транзакции один контракт должен вызвать другой контракт, это происходит как внутренний вызов (internal invocation) в рамках одной транзакции.
  
  ### 5. Отладка.
  Контракты работают так как написаны и только так. Все необходимые функции в контракте **вы должны предусмотреть самостоятельно!** Если контракт был написан и задеплоен с ошибкой, то он навсегда останется в блокчейне с ошибкой. Существует специальный опкод `selfdestruct`, который позволяет уничтожить контракт. **Если вы не предусмотрели в контракте функцию самоуничтожения или отладки, то контракт навсегда останется в блокчейне и его нельзя будет остановить, изменить или убрать!**
  
  Изменить контракт после того как вы его задеплоили можно только если вы предусмотрели функции для изменения и отладки этого контракта.
  
  
 # Приступаем к написанию контракта.
 
 Наш контракт будет представлять из себя токен. Он должен хранить балансы пользователей и позволять им передавать токены с одного баланса на другой.
 
 Написание контракта начинаем с версии компилятора, рекомендуем использовать 0.4.11

```js
pragma solidity ^0.4.11;
```

Далее идет сама структура контракта. Это может являться аналогом функции Main в С++

```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
}
```
Нам необходимо хранить балансы ползователей в своем контракте. Балансы пользователей представляют из себя пары адрес->состояние баланса. Будем использовать структуру `mapping (address => uint) balance;`

`address` - специальный тип для хранения шестнадцатиричных адресов Ethereum.
`uint`    - 256-битное целое беззнаковое число.

Наш контракт теперь будет выглядеть так:
```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
    mapping (address => uint) balance;
}
```
Обратиться к балансу какого-либо пользователя можно так: `balance[0x0123456] = 123;` << Присвоение адресу `0x0123456` некоторого количества токенов (`123`).

Переменные инициализируются в момент создания контракта. Если переменной явно не присвоено значение, то она будет инициализирована со значением стандартным для данного типа (0x0 для адреса, 0 для uint, false для bool).

Существуют локальные переменные - это переменные, объявленные внутри вызова функций. Такие переменные не расходуют **газ** на создание и не записываются в данные хранилища контракта, а существуют только локально в памяти клиента в момент исполнения. По завершении исполнения функции они удаляются.

Т.к. контракт создается транзакцией, то в момент создания контракта доступны переменные `msg`, так что мы можем инициализировать контракт и сразу же присвоить 100 токенов создателю.

```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
    balance[msg.sender] = 100;
    mapping (address => uint) balance;
}
```

Теперь мы должны написать функцию для передачи токенов с баланса одного пользователя на баланс другого. Создадим функцию, которая будет принимать два параметра: количество передаваемых токенов и адрес получателя.

Когда пользователь хочет передать свои токены он должен вызвать функцию в данном контракте и передать в неё параметры: сколько токенов он собирается передать и адрес того кому передаются эти токены.

В результате выполнения мы должны уменьшить баланс отправителя транзакции на указанное количество токенов и увеличить баланс получателя на это же значение.


```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
    balances[msg.sender] = 100;
    mapping (address => uint) balance;
    
    function transfer(uint amount, address recipient) {
        balance[msg.sender] -= amount;
        balance[recipient] += amount;
    }
}
```

В нашем случае контракт содержит потенциальную уязвимость. Если пользователь не имеет на балансе `amount` токенов, то результатом вычитания `amount` из его баланса станет переполнение стека и баланс пользователя станет равен `2**256 - (balance[msg.sender] - amount)`, а не ошибка `Stack Overflow`.

Чтобы этого избежать нужно добавить проверку перед выполнением вычислений.


```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
    balances[msg.sender] = 100;
    mapping (address => uint) balance;
    
    function transfer(uint amount, address recipient) {
        if(balance[msg.sender] < amount) {
          throw;
        }
        balance[msg.sender] -= amount;
        balance[recipient] += amount;
    }
}
```

Опкод `throw` вызовет прерывание транзакции и сожжет весь оставшийся на момент исполнения газ.

Теперь добавим getter функцию чтобы пользователи могли поросматривать балансы без необходимости посылать в контракт транзакцию и тратить на это средства. Добавим к функции модификатор `constant returns` - это позволит вызывать конракт локально для извлечения значения (`return`). Нам так же необходимо указать один параметр - баланс кого мы хотим узнать и тип возвращаемого значения (`uint` т.к. это число токенов на балансе).

```js
pragma solidity ^0.4.11;

contract Token
{
    // COMMENT: This is my token contract
    balances[msg.sender] = 100;
    mapping (address => uint) balance;
    
    function transfer(uint amount, address recipient) {
        if(balances[msg.sender] < amount) {
          throw;
        }
        balance[msg.sender] -= amount;
        balance[recipient] += amount;
    }
    
    function balanceOf(address holder) constant returns (uint) {
      return balance[holder];
    }
}
```

На этом всё, наш контракт простого токена готов. Вы можете задеплоить его в тестовую сеть и попробовать его использовать. Как задеплоить контракт вы можете узнать в этой инструкции: https://github.com/Sparke2/Contract-Tutorials/blob/master/README.md

Вы так же можете прочитать о стандартах токенов для Ethereum: [ERC20 стандарт токена](https://github.com/ethereum/eips/issues/20) и [ERC223 стандарт токен](https://github.com/ethereum/eips/issues/223).
