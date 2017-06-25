---
title: 简易示例：Snack
description: 一个微型应用程序，从我们的GitHunt示例应用程序实现一个视图，您可以直接从浏览器运行和编辑。
---

查看Apollo Client和GraphQL可以为您做的最简单的方法是自己尝试。以下是单个React Native视图的简单示例，它使用Apollo Client与我们的托管示例应用程序[GitHunt](example-schema.md)进行通信。我们已经在[Expo](https://expo.io)中的[Snack](https://blog.expo.io/sketch-a-playground-forrereate-native-16b2401f44a2)编辑器的页面中嵌入了该页面。

<div data-snack-id="HkhGxRFhe" data-snack-platform="ios" data-snack-preview="true" style="overflow:hidden;background:#fafafa;border:1px solid rgba(0,0,0,.16);border-radius:4px;height:514px;width:100%"></div>
<script async src="https://snack.expo.io/embed.js"></script>

首先，让我们来运行应用程序。有两种方法：

1.点击“点击播放”按钮在模拟器中运行应用程序。
2.在[新窗口](https://snack.expo.io/HkhGxRFhe)中打开编辑器，并安装[Expo app](https://expo.io/)以在iOS或Android设备上运行。

无论哪种方式，应用程序将在您键入时自动重新加载。

<h2 id="first-edit">首次编辑</h2>

幸运的是，这个应用是为了让我们的第一个代码变化而设置的。希望在启动应用程序后，您将看到一个具有某些GitHub存储库的可滚动列表的视图。这是使用GraphQL查询从服务器获取的。我们通过删除`stargazers_count`前面的`#`来编辑查询，这样应用程序也会加载GitHub的星号。查询现在应如下所示：

```graphql
{
  feed (type: TOP, limit: 10) {
    repository {
      name, owner { login }

      # 取消注释下面的行以获取星号！
      stargazers_count
    }

    postedBy { login }
  }
}
```

因为在这个例子中，我们已经开发了这样一个UI，它知道如何显示这些新信息，你应该看到应用程序刷新，并显示每个存储库中的GitHub的数量！

<h2 id="github-api">多个后端</h2>

GraphQL最酷的一件事是它可以是多个后端之上的抽象层。您以上执行的查询实际上是从两个完全独立的数据源一次加载的：一个存储提交列表的PostgreSQL数据库，以及用于获取有关实际存储库的信息的GitHub REST API。

让我们验证该应用程序是_actually_从GitHub读取。选择列表中的一个存储库，例如[apollographql/apollo-client](https://github.com/apollographql/apollo-client)或[facebook/graphql](https://github.com/facebook/graphql)，并将其加载。然后，拉下应用程序中的列表以进行刷新。你应该看到数字改变！这是因为我们的GraphQL服务器每次都从真正的GitHub API中提取，并通过一些不错的缓存和ETag处理来避免触发速率限制。

<h2 id="code-explanation">解释代码</h2>

在您离开并使用Apollo构建您自己的令人敬畏的GraphQL应用程序之前，让我们来看看这个简单示例中的代码。

这一段代码使用 `react-apollo` 中的 `graphql` 高阶组件将GraphQL查询结果附加到 `Feed` 组件：

```js
const FeedWithData = graphql(gql`{
  feed (type: TOP, limit: 10) {
    repository {
      name, owner { login }

      # 取消注释下面的行以获取星号！
      # stargazers_count
    }

    postedBy { login }
  }
}`, { options: { notifyOnNetworkStatusChange: true } })(Feed);
```

这是React Native渲染的主要 `App` 组件。它创建一个带有服务器URL的网络接口，初始化一个 `ApolloClient` 实例，并使用 `ApolloProvider` 将其附加到我们的React组件树。如果您使用Redux，这应该是熟悉的，因为它类似于Redux提供程序的工作原理。

```js
export default class App {
  createClient() {
    // 使用URL将Apollo Client初始化到我们的服务器
    return new ApolloClient({
      networkInterface: createNetworkInterface({
        uri: 'http://api.githunt.com/graphql',
      }),
    });
  }

  render() {
    return (
      // 将客户端实例导入到React组件树中
      <ApolloProvider client={this.createClient()}>
        <FeedWithData />
      </ApolloProvider>
    );
  }
}
```

接下来，我们得到一个实际处理一些数据加载问题的组件。大多数情况下，这只是通过 `data` 行为传递给 `feedList`，这实际上会显示这些项目。但是，这里还有一个有趣的React Native `RefreshControl`组件，当您拉下列表时，它使用Apollo内置的 `data.refetch` 方法来提取数据。它还使用 `data.networkStatus` 支架显示正确的加载状态，Apollo为我们追踪。

```js
// 这里的数据资料来自阿波罗HoC。它有我们要求的数据，还有一些有用的方法，如 refetch().
function Feed({ data }) {
  return (
    <ScrollView style={styles.container} refreshControl={
      // 这样可以实现拉拔刷新功能
      <RefreshControl
        refreshing={data.networkStatus === 4}
        onRefresh={data.refetch}
      />
    }>
      <Text style={styles.title}>GitHunt</Text>
      <FeedList data={data} />
      <Text style={styles.fullApp}>See the full app at www.githunt.com</Text>
      <Button
        buttonStyle={styles.learnMore}
        onPress={goToApolloWebsite}
        icon={{name: 'code'}}
        raised
        backgroundColor="#22A699"
        title="Learn more about Apollo"
      />
    </ScrollView>
  );
}
```

最后，我们到达实际显示项目 `FeedList` 组件的位置。这消耗了阿波罗的 `data` 支柱，并将其映射到显示列表项。您可以看到，由于GraphQL，我们得到的数据完全符合我们预期的形状。

```js
function FeedList({ data }) {
  if (data.networkStatus === 1) {
    return <ActivityIndicator style={styles.loading} />;
  }

  if (data.error) {
    return <Text>Error! {data.error.message}</Text>;
  }

  return (
    <List containerStyle={styles.list}>
      { data.feed.map((item) => {
          const badge = item.repository.stargazers_count && {
            value: `☆ ${item.repository.stargazers_count}`,
            badgeContainerStyle: { right: 10, backgroundColor: '#56579B' },
            badgeTextStyle: { fontSize: 12 },
          };

          return <ListItem
            hideChevron
            title={`${item.repository.owner.login}/${item.repository.name}`}
            subtitle={`Posted by ${item.postedBy.login}`}
            badge={badge}
          />;
        }
      ) }
    </List>
  )
}
```

现在，您已经看到了构建React Native应用程序所需的所有代码，该应用程序加载了项目列表，并将其显示在具有拉到刷新功能的列表中。正如你所看到的，我们不得不写很少的数据加载代码！我们认为Apollo和GraphQL可以帮助数据加载脱离您的方式，以便您可以比以前更快地构建应用程序。

<h2 id="next-steps">下一步</h2>

让我们从头开始构建你自己的应用程序！您有两个教程可以通过，我们建议按以下顺序进行操作：

1. [Full-Stack GraphQL + React tutorial](https://dev-blog.apollodata.com/full-stack-react-graphql-tutorial-582ac8d24e3b#.cwvxzphyc) 来自 [Jonas Helfer](https://twitter.com/helferjs)，Apollo Client的首席开发人员。
2. 由[Graphcool](https://www.graph.cool/)团队和社区开发的[Learn Apollo](https://www.learnapollo.com), Graphcool是一个后端托管平台。

玩的开心！