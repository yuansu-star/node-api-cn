<!-- YAML
added: v0.1.21
changes:
  - version: v10.2.0
    pr-url: https://github.com/nodejs/node/pull/20485
    description: The `error` parameter can be an object containing regular
                 expressions now.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/17584
    description: The `error` parameter can now be an object as well.
  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3276
    description: The `error` parameter can now be an arrow function.
-->
* `fn` {Function}
* `error` {RegExp|Function|Object|Error}
* `message` {string}

期望 `fn` 函数抛出错误。

如果指定，则 `error` 可以是 [`Class`]、[`RegExp`]、验证函数，每个属性将被测试严格的深度相等的验证对象、或每个属性（包括不可枚举的 `message` 和 `name` 属性）将被测试严格的深度相等的错误实例。
使用对象时，还可以在对字符串属性进行验证时使用正则表达式。
请参阅下面的示例。

如果指定 `message`，则当 `fn` 调用无法抛出或错误验证失败时，`message` 将附加到 `AssertionError` 提供的消息。

自定义的验证对象/错误实例：

```js
const err = new TypeError('错误值');
err.code = 404;
err.foo = 'bar';
err.info = {
  nested: true,
  baz: 'text'
};
err.reg = /abc/i;

assert.throws(
  () => {
    throw err;
  },
  {
    name: 'TypeError',
    message: '错误值',
    info: {
      nested: true,
      baz: 'text'
    }
    // 仅测试验证对象上的属性。
    // 使用嵌套对象需要存在所有属性。
    // 否则验证将失败。
  }
);

// 使用正则表达式验证错误属性：
assert.throws(
  () => {
    throw err;
  },
  {
    // `name` 和 `message` 属性是字符串，使用正则表达式将匹配字符串。 
    // 如果失败，则会抛出错误。
    name: /^TypeError$/,
    message: /错误/,
    foo: 'bar',
    info: {
      nested: true,
      // 无法对嵌套属性使用正则表达式！
      baz: 'text'
    },
    // `reg` 属性包含一个正则表达式，
    // 并且只有当验证对象包含相同的正则表达式时，
    // 它才会通过。
    reg: /abc/i
  }
);

// 由于 `message` 和 `name` 属性不同而失败：
assert.throws(
  () => {
    const otherErr = new Error('未找到');
    otherErr.code = 404;
    throw otherErr;
  },
  err // 测试 `message`、`name` 和 `code`。
);
```

使用构造函数验证 instanceof：

```js
assert.throws(
  () => {
    throw new Error('错误值');
  },
  Error
);
```

使用 [`RegExp`] 验证错误消息：

使用正则表达式在错误对象上运行 `.toString`，因此也将包含错误名称。

```js
assert.throws(
  () => {
    throw new Error('错误值');
  },
  /^Error: 错误值$/
);
```

自定义的错误验证函数：

```js
assert.throws(
  () => {
    throw new Error('错误值');
  },
  (err) => {
    assert(err instanceof Error);
    assert(/value/.test(err));
    // 除了 `true` 之外，不建议从验证函数返回任何内容。 
    // 这样做会导致再次抛出捕获的错误。 
    // 这通常不是理想的结果。 
    // 抛出有关失败的特定验证的错误（如本例所示）。
    return true;
  },
  '不是期望的错误'
);
```

`error` 不能是字符串。
如果提供了一个字符串作为第二个参数，则假定 `error` 被忽略，而字符串将用于 `message`。
这可能导致容易错过的错误。
使用与抛出的错误消息相同的消息将导致 `ERR_AMBIGUOUS_ARGUMENT` 错误。
如果使用字符串作为第二个参数，请仔细阅读下面的示例：

<!-- eslint-disable no-restricted-syntax -->
```js
function throwingFirst() {
  throw new Error('错误一');
}
function throwingSecond() {
  throw new Error('错误二');
}
function notThrowing() {}

// 第二个参数是一个字符串，输入函数抛出一个错误。
// 第一种情况不会抛出，因为它与输入函数抛出的错误消息不匹配！
assert.throws(throwingFirst, '错误二');
// 在下一个示例中，传入的消息类似来自错误的消息，
// 并且由于不清楚用户是否打算实际匹配错误消息，
// 因此 Node.js 抛出了 `ERR_AMBIGUOUS_ARGUMENT` 错误。
assert.throws(throwingSecond, '错误二');
// TypeError [ERR_AMBIGUOUS_ARGUMENT]

// 该字符串仅在函数未抛出时使用（作为消息）：
assert.throws(notThrowing, '错误二');
// AssertionError [ERR_ASSERTION]: Missing expected exception: 错误二

// 如果要匹配错误消息，请执行以下操作：
// 它不会抛出错误，因为错误消息匹配。
assert.throws(throwingSecond, /错误二$/);

// 如果错误消息不匹配，则不会捕获函数内的错误。
assert.throws(throwingFirst, /错误二$/);
// Error: 错误一
//     at throwingFirst (repl:2:9)
```

由于令人困惑的表示法，建议不要使用字符串作为第二个参数。
这可能会导致难以发现的错误。

