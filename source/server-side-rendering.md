---
title: 服务端渲染
---


Apollo提供两种技术，可让你的应用程序快速加载，避免不必要的延迟给用户：

- Store 覆水，允许你的初始查询集立即返回数据，而无需服务器往返。
- 服务端渲染，它将服务器上的初始HTML视图发送给客户端。

你可以使用这些技术中的一种或两种来提供更好的用户体验。

<h2 id="store-rehydration">Store 覆水</h2>

对于在客户机上呈现UI之前可以在服务器上执行某些查询的应用程序，Apollo允许设置数据的初始状态。这有时被称为覆水，因为数据被序列化并包含在初始HTML有效载荷中时，数据被“脱水”。

例如，一个典型的方法是包括一个脚本标签，看起来像这样：

```html
<script>
  // `initialState` 应该具有Appollo存储状态的形状。确保只包括数据，例如：
  // const initialState = {[client.reduxRootKey]: {
  //   data: client.store.getState()[client.reduxRootKey].data
  // }};
  window.__APOLLO_STATE__ = initialState;
</script>
```

然后可以使用从服务器传递的初始状态来重新使用客户端：
```js
const client = new ApolloClient({
  initialState: window.__APOLLO_STATE__,
});
```

我们将在下面看到如何使用Node和`react-apollo`的服务器渲染功能生成HTML和Apollo商店的状态。但是，如果你通过其他方式呈现HTML，则必须手动生成该状态。

如果你在Apollo外部使用[Redux](redux.html)，并且已经具有存储补液，则应将存储状态传递到[`Store`构造函数](http://redux.js.org/docs/basics/Store.html)。

然后，当客户端运行第一组查询时，数据将立即返回，因为它已经在存储中！

如果你在某些初始查询中使用[`forceFetch`](cache-updates.html#forceFetch)，则可以在初始化期间传递 `ssrForceFetchDelay` 选项来跳过强制提取，以便即使这些查询也使用缓存运行：

```js
const client = new ApolloClient({
  initialState: window.__APOLLO_STATE__,
  ssrForceFetchDelay: 100,
});
```

<h2 id="server-rendering">服务端渲染</h2>

你可以使用`react-apollo`中内置的渲染函数在Node服务器上渲染整个基于React的Apollo应用程序。这些功能负责提取渲染组件树所需的所有查询。通常，你可以在HTTP服务器（如[Express](https://expressjs.com)）中使用这些功能。

不需要更改客户端查询来支持此操作，因此你的基于Apollo的React UI应支持SSR开箱即用。

<h3 id="server-initialization">服务端初始化</h3>

为了在服务器上呈现应用程序，你需要处理HTTP请求（使用像Express这样的服务器和支持服务器的路由器，如React-Router），然后将应用程序渲染为一个字符串以传回响应。

我们将在下一部分中看到如何使用组件树并将其转换为一个字符串，但是你需要对如何在服务器上构建Apollo Client实例有一点小心，以确保一切都在此处工作：

1. 当服务器上的[创建Apollo Client实例](initialization.html)时，需要设置网络接口才能正确连接到API服务器。这可能与你在客户端上的操作看起来有所不同，因为如果你在客户端上使用相对URL，则可能需要使用绝对URL到服务器。

2. 由于你只想获取每个查询结果一次，请将`ssrMode: true`选项传递给Apollo Client构造函数，以避免重复强制提取。

3. 你需要确保为每个请求创建一个新的客户端或存储实例，而不是为多个请求重新使用相同的客户端。否则，用户界面将变得过时，你将会遇到[authentication](auth.html)的问题。

一旦你把它们放在一起，你会得到如下初始化代码：

```js
import { ApolloClient, createNetworkInterface, ApolloProvider } from 'react-apollo';
import Express from 'express';
import { match, RouterContext } from 'react-router';

// 路由文件是客户端和服务器之间良好的共享入口点
import routes from './routes';

// 请注意，你不必使用任何特定的http服务器，但是我们在这个例子中使用Express
const app = new Express();
app.use((req, res) => {

  // 此示例使用React Router，尽管它应该与支持SSR的其他路由器同样好
  match({ routes, location: req.originalUrl }, (error, redirectLocation, renderProps) => {

    const client = new ApolloClient({
      ssrMode: true,
      // 请记住，这是SSR服务器将用于连接到API服务器的接口，因此我们需要确保它不被防火墙等
      networkInterface: createNetworkInterface({
        uri: 'http://localhost:3010',
        opts: {
          credentials: 'same-origin',
          headers: {
            cookie: req.header('Cookie'),
          },
        },
      }),
    });

    const app = (
      <ApolloProvider client={client}>
        <RouterContext {...renderProps} />
      </ApolloProvider>
    );

    // 渲染代码（见下文）
  });
});

app.listen(basePort, () => console.log( // eslint-disable-line no-console
  `App Server is now running on http://localhost:${basePort}`
));
```
你可以查看[GitHunt应用程序的`ui/server.js`](https://github.com/apollographql/GitHunt-React/blob/master/ui/server.js) 以获取完整的工作示例。

接下来我们将看看渲染代码实际上是什么。

<h3 id="getDataFromTree">使用 `getDataFromTree`</h3>

`getDataFromTree` 方法使用React树，确定需要哪些查询来渲染它们，然后将其全部获取。如果你有嵌套查询，它会递归地放在整个树上。它返回一个承诺，当你的Apollo Client存储中的数据准备就绪时可以解决。

在承诺解决之前，你的Apollo Client存储将被完全初始化，这意味着你的应用程序现在将立即呈现（因为所有查询都被预取），并且你可以在响应中返回字符串结果：

```js
import { getDataFromTree } from "react-apollo"

const client = new ApolloClient(....);

// 请求（见上文）
getDataFromTree(app).then(() => {
  // 我们准备好渲染真实的DOM
  const content = ReactDOM.renderToString(app);
  const initialState = {[client.reduxRootKey]: client.getInitialState()  };

  const html = <Html content={content} state={initialState} />;

  res.status(200);
  res.send(`<!doctype html>\n${ReactDOM.renderToStaticMarkup(html)}`);
  res.end();
});
```

在这种情况下，你的标记可以看起来像这样：

```js
function Html({ content, state }) {
  return (
    <html>
      <body>
        <div id="content" dangerouslySetInnerHTML={{ __html: content }} />
        <script dangerouslySetInnerHTML={{
          __html: `window.__APOLLO_STATE__=${JSON.stringify(state).replace(/</g, '\\u003c')};`,
        }} />
      </body>
    </html>
  );
}
```

<h3 id="local-queries">避免本地查询的网络</h3>

如果你的GraphQL端点位于与你呈现的同一台服务器上，则可能希望在进行SSR查询时避免使用网络。特别是，如果本地主机在你的生产环境（例如Heroku）上进行防火墙，则会对这些查询进行网络请求不起作用。该问题的一个解决方案是[apollo-local-query](https://github.com/af/apollo-local-query) 模块，它允许你为apollo创建一个实际不使用的 `networkInterface` 网络。


If your GraphQL endpoint is on the same server that you're rendering from, you may want to avoid using the network when making your SSR queries. In particular, if localhost is firewalled on your production environment (eg. Heroku), making network requests for these queries will not work. One solution to this problem is the [apollo-local-query](https://github.com/af/apollo-local-query) module, which lets you create a `networkInterface` for apollo that doesn't actually use the network.

<h3 id="skip-for-ssr">跳过SSR查询</h3>

如果要在SSR期间有意地跳过查询，可以在查询选项中传递`ssr: false`。通常，这将意味着组件将在服务器上处于加载状态。例如：

```js
const withClientOnlyUser = graphql(GET_USER_WITH_ID, {
  options: { ssr: false }, // 不会在SSR期间调用
});
```

<h3 id="renderToStringWithData">使用 `renderToStringWithData`</h3>

`renderToStringWithData` 方法简化了上面的内容，只需返回需要渲染的内容字符串即可。所以它略微减少了步数：

```js
// 服务器应用程序代码（集成使用）
import { renderToStringWithData } from "react-apollo"

const client = new ApolloClient(....);

// 在请求期间
renderToStringWithData(app).then((content) => {
  const initialState = {[client.reduxRootKey]: client.getInitialState() };
  const html = <Html content={content} state={initialState} />;

  res.status(200);
  res.send(`<!doctype html>\n${ReactDOM.renderToStaticMarkup(html)}`);
  res.end();
});
```
