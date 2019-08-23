- 为什么要创建这个repo呢？

> 想写一篇文章扫盲，分析一下基于nodejs做项目都有哪些自动化测试难点，堵住那些因不会写而放弃写测试的人的嘴
>
> 再就是自己用nodejs做项目时间长了（1年了），感觉还是有点心得的
>
> 透露一下现在项目的用nodejs写的服务端应用，测试覆盖率的四个维度branch、function、line还有statement都在80%左右徘徊，并且因为ci上丧心病狂的coverage check，我们的测试覆盖率还在稳步提升

- 大致会包含什么内容呢？

> 推荐一些测试工具组合，比如jest，mocha，mocha cake，nyc，chai……
>
> 推荐一些疑难测试的破解方案
>
> 列举一些自动化测试的反模式
>
> 强推能很快提升测试覆盖率的实践：TDD

- 不想包含什么内容？

> 不想讨论各种互有overlapping的测试名称定义
> 
> 不想讨论为啥要写测试
> 
> 不想讨论TDD到底好不好
> 
> 不会涉及如何针对Angular，React，Vue做自动化测试

## 初窥门径

既然叫automatic test guide，那自然要从最简单的纯函数测试讲起。

为什么这种测试最简单呢？因为待测试的是纯函数（pure function）。

为什么待测试的是纯函数就很简单呢？让我们来复习一下javascript里纯函数的定义。

#### [纯函数](https://en.wikipedia.org/wiki/Pure_function)
- 接受相同的输入时一定能产生相同的输出
    - 函数的输出值**只可以**与入参有关
- 不能有副作用
    - 副作用包含改变引用类型```reference arguments```入参，触发事件，触发输出设备执行I/O输出等
    - 允许修改基本类型的入参

> 基本类型 [boolean, null, undefined, String, number] 
>
> 引用类型 [Array, Function, Object]

那么，接下来要上测试了，测试体将使用given，when，then三部曲来撰写。如果你还没听过这三部曲，建议移步[这里](https://martinfowler.com/bliki/GivenWhenThen.html)了解一下。

```javascript
it('should be able to calculate total price according to unit price and quantity', () => {
  //given
  const quantity = 5;
  const unitPrice = 1;
  
  //when
  const totalPrice = calculateTotalPrice(quantity, unitPrice);
  
  //then
  expect(totalPrice).toEqual(5);
})
```

> 为了省事儿，之后的示例只会用空行分隔given，when，then而不再写注释

这个测试的一个实现可以是
```javascript
const calculateTotalPrice = (quantity, unitPrice) => quantity * unitPrice
```

纯函数的应用，十分宽广。

做一个布尔值判断，执行一次乘法运算，转换一下第三方接口拿到的response，各种util……不一而足

但是即使是这样的纯函数，其测试也有变得复杂起来的时候。

假如totalPrice需要从一个第三方API的response中计算出，那么这个测试可能变成这样

```javascript
it('should be able to calculate total price according to price response', () => {
  const priceResponse = {
    supermarket: [
      {
        name: '牙刷',
        unitPrice: 3,
        quantity: 1
      },
      ...
    ]
    '7-11': [
      {
        productName: '维他柠檬茶',
        price: 6.5
      },
      ...
    ],
    ...
  };
  
  const totalPrice = calculateTotalPrice(priceResponse);
  
  expect(totalPrice).toEqual(3286546273545326.00);
})
```

没错，只把准备数据从两个基础类型变量变成一大坨Object，就能让这个测试看起来十分臃肿。

这种时候，我的建议是不要把这一大坨准备的数据放在测试体里，你可以把这一大坨准备数据变成JSON格式存在一个称为```Fixture```的文件夹下。然后从测试体里引用这个JSON文件即可。

> Fixture一般存放测试数据集

这时候这个测试就会变成这样

```javascript
it('should be able to calculate total price according to price response', () => {
  const priceResponse = require('../Fixture/price-response.json');
  
  const totalPrice = calculateTotalPrice(priceResponse);
  
  expect(totalPrice).toEqual(3286546273545326.00);
})
```

是不是感觉一下子烦恼尽消，犹如夏日炎炎挥汗如雨时喝到了一瓶冰冻过的维他柠檬茶？

但是这样做，有可能还是有问题。当你有多个测试用例需要共享一部分测试数据集的时候如果还是分JSON存储，那你很容易把KPI（假如KPI和代码量挂钩的话><）做上去，为啥啊，太多重复的大JSON文件了呗。

测试代码，也请像对待产品代码一样对待，如果有重复的测试数据集，去重也是必要的。

你可以抽取一些创建测试数据的函数，可以用一些工厂方法来产生测试数据，还可以写query从测试数据集的JSON里pick你想要的数据。

但是有一点规则请遵守，那就是不要过于吝啬过于精简你的这些为造数据而产生的函数们。因为测试的一大作用就是充当文档，让读者快速了解到一些重要变量在runtime时长什么样。没有人喜欢读晦涩难懂的文档。

在我看来，测试的可读性是稍大于其代码的整洁性的。这个平衡每个团队都不一样，希望读者们都能找到自己团队中大家都能接受的平衡点。

下面请看一个具有明显反模式的测试用例

```javascript
it('should be able to calculate prices according to price response', () => {
  const rawPriceResponse = require('../Fixture/price-response.json');
  const priceResponse = _.chain(rawPriceResponse)
    .values()
    .flatMap((products) => _.map(products, (product) => ({
        ...product,
        quantity: product.quantity || 0
      }))
    )
    .filter({price: 0})
    .value();
    
  const prices = calculatePrices(priceResponse);
  
  expect(prices).toEqual(...);
})

```

接下来，请看一下这个测试

```javascript
it('should be able to calculate prices according to price response', () => {
  const priceResponse = require('../Fixture/price-response.json');
  
  const prices = calculatePrices(priceResponse);
  
  expect(prices).toEqual({
    '7-11': {
      averagePrice: 10.23,
      mostPopularPrice: 8.8,
      leastPopularPrice: 4.4,
      totalPrice: 1273512.27,
      ...
    },
    supermarket: {
      weekdayAveragePrice: 10.8,
      weekendAveragePrice: 14.1,
      ...
    },
    ...
  });
})
```

这是另外一种极端情况，这个测试的断言十分冗长，甚至可能冗长到你一屏幕都看不下这断言的代码。

这种时候，除了可以把expected的result也变成JSON存到Fixture里之外，还可以使用以下办法来让测试看起来更优雅

- 拆分测试case分模块断言
- 选取代表性数据做取样精确断言
- 模糊断言

一个单元测试里涵盖太多东西不见得是好事，因为这样会带来因冗长而阅读困难，关注点分散这样的坏味道。

还是用上面的例子，如果拆分case的话，至少可以拆成两个测试case。第一个只准备```7-11```的计算数据，使得算出来的结果也只包含```7-11```的数据，结果集大幅减少。第二个测试则只注重于supermarket的数据准备和结果断言。

选取代表性数据做抽样断言，例如可以认为```7-11```的数据结构十分具有代表性，一旦```7-11```的结果算对了，那么别的也都能算对。所以可以只断言```7-11```的结果是正确的来保证业务逻辑。

> 不得不说要做这样的决定实在需要魄力，因为很容易在code review中被challenge为什么不做精确的全量断言。

模糊断言是什么呢，模糊断言一般会去断言这个结果的长度，是否拥有某一属性，某属性下的值是否正确，结果的类型是否为期望的类型……
当结果十分冗长且非常易于改变时，我推荐这样的断言，因为这能让测试不会成为生产代码的复制品。

例如拿到第三方集成系统给的response去解析的时候，只要断言一下基本结构和基本字段存在并且类型正确就足够了。不需要管它里面的数据正确与否。因为我不为它们数据的正确性做保障。

需要警惕的是，假如这种时候精准断言了，它们返回结构不变数据却改变了，那也会使测试红掉。

#### 小结

纯函数状态易于预测，构建测试十分方便。纯函数的测试实在是居家旅行，减少加班，必备良药喔。

请各位对号入座，如果你项目上的纯函数测试覆盖率连80%都还没达到的话，也不用继续往下看了。赶紧去补纯函数的测试。

## 小有成就

这里就要开始介绍非纯函数的测试方法了。
nodejs的代码，很常见的一点就是，需要连接数据库。那么自然需要对一些包含数据库交互操作的函数进行测试。

这里先把结论写了，那就是

#### 轻量级数据库操作不隔离，重量级数据库操作酌情隔离，尽量不使用和生产环境不一样的数据库作为测试数据库。

我时常会看到一些项目，为了加快测试的运行速度，在跑测试的时候使用内存数据库来代替真实的产品环境数据库。
我不建议这样做。原因在于，这样的测试会暴露只在内存数据库上发生的问题，会掩盖产品数据库上可能发生的问题。
测试通过了也并不能提供充足的安全保障，不能给予团队充足的信心，这样的测试，不要也罢。

那么，在跑测试的时候连接数据库，又会是怎么样的呢？
首先会在测试入口处连接一下测试数据库，然后就像写生产代码那般自然地在准备测试数据时调用带有数据库交互的函数来写入准备数据。
请注意我这里用了```就像写生产代码那般自然```这样的描述。
这意味着准备数据的语句可以直接从生产代码抄过来，嗯，```copy and paste from ***```这种万能手法想必各位早已精通。

所以轻量级数据库操作的测试用例，有可读性高，运行速度快，写起来比隔离简单这三个优点。

接下来还有一件重要的事情，既然测试都动用到了数据库，那么就很有必要为```脏数据```善后。
对于待测功能的测试代码，这些数据是```准备数据```，然而对于别的测试代码，这些数据就会成为```脏数据```。
优雅的我们自然会在这样的测试断言后去优雅地删除曾经插入过的准备数据。

> 如果没有及时清除脏数据，就很有可能发生某个测试在单独运行是绿的，所有测试一起运行时却会变红的灵异现象。

下面请看一个测试带有数据库交互的测试用例

```javascript
TODO 这里要放一个直接调用数据库的测试用例噢><
     这个例子不要使用beforeEach等hook
```

我将之称之为轻量级数据库操作，原因就是除开连接数据库的时间，插入数据和删除数据所消耗的时间都是毫秒级的，在整套测试用率的运行中可以忽略不计。

那么重量级数据库操作的测试怎么办呢？```酌情```而定是否隔离。在测试一些并不那么致命，较为边缘的重数据库操作时，使用```stub```对真实的数据库操作进行隔离。

就像这样

```javascript
TODO 这里需要一个不那么致命，边缘的重量级数据库操作测试用例哟。 
```

> 如果对类似stub这样的```测试替身```不熟悉，请看看[这儿](https://martinfowler.com/bliki/TestDouble.html)。

在测试一些致命分支，主线功能，核心函数时，其实重量级数据库操作的测试也并非洪水猛兽。
有一个很简单的判断方法，假如对产品代码进行了测试后，整个团队的信心都有显著提升。
那也能接受测试里包含时间消耗稍长的重量级数据库操作。

当然这个也很玄乎。可接受时间视项目而定，重量级数据库操作所消耗时间和团队信心提升的平衡如何找到就是另一门艺术了。

> 写一些重量级数据库操作测试用例时，别忘了把数据量大的时候把需要准备的数据移到```Fixture```下。请时刻警惕使可读性变糟糕的坏味道。

最后我还想提一些关于提升测试可读性和优化测试速度的建议。

假如多个测试用例需要用到同一套测试数据，那么可以把准备数据放到```describe```下的```beforeEach```中。这时候请把数据清理的语句移动到```afterEach```

> 你猜，如果把beforeEach放在describe外会咋样。

假如并不是所有用例都需要用到```beforeEach```或者```afterEach```中的准备数据，要么把测试用例重新组织移出这个```describe```，要么把准备数据的语句移到具体的测试用例中。

> 请确保不要让你的测试在运行的时候进行一些无谓的数据准备，或者额外的数据清理，那有可能是极大的消耗。

#### 小结

假如涉及到网络调用，其处理方式跟连接数据库的测试大同小异。测试怎么做，有经验的人很快能找准平衡点让团队满意，没有经验的话就只能慢慢摸索了。掌握好以上测试套路的利弊会为你找平衡点做决策提供坚实的理论基础。

## 炉火纯青

走到这一步，你已经是一个能打的Javascript自动化测试开发工程师了。

你需要学会应对掺杂着随机因素的测试用例。

随机因素可以是即时时间，即时生成的随机数和即时生成的uuid。
这样的测试因为输入不是确定的，所以我们需要把不确定性变为确定性再做测试。

有三种套路可以处理这种情况。

1. 从待测函数中剥离随机因素

> 实现代码是这样的
```javascript
const uuid = require('uuid');
const generateOrder = (order) => ({ orderId: uuid(), ...order });
const order = generateOrder({ name: 'Lemon Tea', unitPrice: 20 });
```

> 先做一点重构变成这样
```javascript
const uuid = require('uuid');
const generateOrder = (order, uuid) => ({ orderId: uuid, ...order });
const order = generateOrder({ name: 'Lemon Tea', unitPrice: 20 }, uuid());
```

> 测试就能这样写了
```javascript
it('should be able to generate an order', () => {
  const orderInfo = { name: 'Lemon Tea', unitPrice: 20 };
  const uuid = uuid();
  
  const order = generateOrder(orderInfo, uuid);
  
  expect(order).toEqual({
    orderId: uuid,
    name: 'Lemon Tea',
    unitPrice: 20
  });
})
```

这样写测试的话，生成随机因素的语句或者逻辑往往极度简单，测试时把随机因素剥离了，就变成测试纯函数了。
假如剥离掉随机因素后会让你对测试的信心不足，那么请继续往下看。

2. 使用mock隔离随机因素

这是一个把随机变稳定的套路。好处在于我们可以正常对所有output做精确断言，提供的信心最高。
```javascript
jest.mock('uuid');
const uuid = require('uuid');
uuid.mockImplementation(() => `uuid-${uuid.mock.calls.length}`);

it('should be able to generate an order', () => {
  const orderInfo = { name: 'Lemon Tea', unitPrice: 20 };
  
  const order = generateOrder(orderInfo);
  
  expect(order).toEqual({
    orderId: 'uuid-1',
    name: 'Lemon Tea',
    unitPrice: 20
  });
})
```

3. 剔除掉随机因素，对结果做部分断言

这个套路就很容易理解了，随机部分不断言就行了。
```javascript
it('should be able to generate an order', () => {
  const orderInfo = { name: 'Lemon Tea', unitPrice: 20 };

  const order = generateOrder(orderInfo);

  delete order.orderId;
  expect(order).toEqual({
    name: 'Lemon Tea',
    unitPrice: 20
  });
})
```

这样的做法有个小问题，就是当随机部分很多的时候，你可能需要写一个循环或者做很多事情才能把随机部分从结果中剥离开来。

**插曲**
> Kevin你瞎说，你上面举例的测试都那么简单，真正做项目的时候哪有那么简单的事情。 你这讲的都是"道理我都懂，但是臣妾做不到啊"。
> 真正的项目上都是一个function一两百行，写个测试得准备几小时数据，一天才多少工作时间，都写不了几个测试。

这就简单了，解决办法就是开发过程中我们需要用TDD。用TDD写出来的代码一定是带测试的，也就破除了难加测试的点。

事后补测试并不是一件好事情，可能是因为代码出bug了，然后需要针对bug数据加一个测试来覆盖保证以后不再重犯。
但是当你想为一堆以前从没有测试覆盖的代码里加上测试的时候，你得去通读代码理解每一个分支每一个细节，然后根据理解把每一个分支都加测试予以覆盖。这意味着什么呢？你可能因为需要知道实现的历史而去询问很多人，你可能需要为了整理代码画一些流程图，这样下来确实一天补不了几个测试。

## 一代宗师

##### TODO
- [ ] 我需要定义一下这个境界是怎么样的
    - [ ] 能推动TDD开发的人
    