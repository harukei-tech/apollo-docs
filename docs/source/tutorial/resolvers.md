---
title: "3. Graph の resolver を作成する"
description: Learn how a GraphQL query fetches data
---

Time to accomplish: _15 Minutes_

ここまでの作業でつくってきた graph API は、まだ機能的に十分とはいえません。graph schema がどうなっているかをチェックしたりすることはできるようになったものの（たとえば playground を使って schema を調べたりすることはできるようにはなっているが）、この graph API に対して query を投げて実行させるということはしていないし、できないからです。なのでここからは graph API の resolver を作成し、今まで頑張って作ってきた data sources を resolver の中で呼び出し、ビジネスロジックをトリガーしたりデータを取得したり更新させたりといった機能を作っていきます。

> Up until now, our graph API hasn't been very useful. We can inspect our graph's schema, but we can't actually run queries against it. Now that we've built our schema and data sources, it's time to leverage all of our hard work by calling our data sources in our graph API's resolver functions to possibly trigger business logic and/or to fetch and/or update data.

## What is a resolver?

**Resolvers** は GraphQL の operation（query や mutation や subscription）が、実際にどのような処理を行なってデータを返すのかという指示書です。**Resolver** は schema で定義した型の値を返さなければいけません。もしくはその型の値の promise を返さなければいけません。

> **Resolvers** provide the instructions for turning a GraphQL operation (a query, mutation, or subscription) into data. They either return the same type of data we specify in our schema or a promise for that data.

> 訳注：今まで作ってきた schema は、Graph API に対してどのような処理ができて、どのような型の値が返ってくるかという設計図だった。しかしこの設計図には、どのように値を取得するのかといった具体的なことは書かれていない。その実際の値を取得したりという具体的な指示は、Resolver に記述する。例えば schema の Query.launches が実行された場合にはそれに対応する Resolver の Query.launches が実行される。Resolver の Query.launches には Context から与えられた Data source を通して API で値をフェッチする処理、それから必要な形への加工等が記述されている。このように schema で全体の構造を設計し、それに対応する resolver で実際に値を取得したり加工したりしたものを返す、という役割分担になっている。

Resolver の開発を始める前に、resolver 関数がどのような形式なのかまず確認していきましょう。Resolver function は四つの引数を受け取ります。

> Before we can start writing resolvers, we need to learn more about what a resolver function looks like. Resolver functions accept four arguments:


```js
fieldName: (parent, args, context, info) => data;
```

- **parent**: 親 resolver から受け取ったオブジェクト。
- **args**: この field に対して渡された引数。
- **context**: GraphQL operation の resolver 全体で共有されるオブジェクト。このチュートリアルでは認証情報やデータソースを context で共有している。
- **info**: 実行したオペレーションに関する状態等の詳細情報。通常は用いられないがよりアドバンスなケースにおいて使われることが多い。

> - **parent**: An object that contains the result returned from the resolver on the parent type
> - **args**: An object that contains the arguments passed to the field
> - **context**: An object shared by all resolvers in a GraphQL operation. We use the context to contain per-request state such as authentication information and access our data sources.
> - **info**: Information about the execution state of the operation which should only be used in advanced cases

 `LaunchAPI`　と `UserAPI` データソースを `ApolloServer` に `context` として渡していたことを思い出してください。その操作をしたこによって、resolver の `context` 引数を通して、これらにアクセすることが可能になっています。

> Remember the `LaunchAPI` and `UserAPI` data sources we created in the previous section and passed to the `context` property of `ApolloServer`? We're going to call them in our resolvers by accessing the `context` argument.

これらの context の説明は今の時点は少しわかりにくいかもしれませんが実際の例をみていくと感覚がつかめるはずです。では実際にみていきましょう！

> This might sound confusing at first, but it will start to make more sense once we dive into practical examples. Let's get started!

### Connecting resolvers to Apollo Server

まずは resolver を Apollo server に組み込みましょう。この時点では resolver は空のオブジェクトにすぎませんが、先に `ApolloServer` に渡してしまいましょう。`src/index.js` に以下のコードを複写します。

> First, let's connect our resolver map to Apollo Server. Right now, it's just an empty object, but we should add it to our `ApolloServer` instance so we don't have to do it later. Navigate to `src/index.js` and add the following code to the file:

_src/index.js_
```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');
const { createStore } = require('./utils');
const resolvers = require('./resolvers'); // highlight-line

const LaunchAPI = require('./datasources/launch');
const UserAPI = require('./datasources/user');

const store = createStore();

const server = new ApolloServer({
  typeDefs,
  resolvers, // highlight-line
  dataSources: () => ({
    launchAPI: new LaunchAPI(),
    userAPI: new UserAPI({ store })
  })
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

これだけで Apollo server は自動的に `launchAPI` と `userAPI` を resolver の context に追加してくれます。

> Apollo Server will automatically add the `launchAPI` and `userAPI` to our resolvers' context so we can easily call them.

## Write Query resolvers

では最初に `launches` と `launch` と `me` のための resolver を書きましょう。これらは `Query` type に所属する filed です。Resolver を書く際には、schema に書いた type や field と一致するキーの場所に resolver を作成する必要があります。（訳注：例えば schema で定義した `Query.me` に対応する resolver は、resolver の `Query.me` の位置に置く必要がある。） どのフィールドがどこにあるのかわからなくなったら graph API schema を見直しましょう。

> First, let's start by writing our resolvers for the `launches`, `launch`, and `me` fields on our `Query` type. We structure our resolvers into a map where the keys correspond to the types and fields in our schema. If you ever get stuck remembering which fields are on a type, you can always check your graph API's schema.

`src/resolvers.js` に移動して以下のコードを複写しましょう。

> Navigate to `src/resolvers.js` and paste the code below into the file:

_src/resolvers.js_

```js
module.exports = {
  Query: {
    launches: (_, __, { dataSources }) =>
      dataSources.launchAPI.getAllLaunches(),
    launch: (_, { id }, { dataSources }) =>
      dataSources.launchAPI.getLaunchById({ launchId: id }),
    me: (_, __, { dataSources }) => dataSources.userAPI.findOrCreateUser()
  }
};
```

上記のコードは、`Query` type filed に対する resolver 関数です。それぞれ `launches` と `launch` と `me` に対する resolver 関数を定義しました。_top-level_ resolver（訳注：Query, Mutation 等 Operation 直下にある特殊な resolver のこと）の `parent` は常に空になります。なぜなら graph のルートに当たるため、親が存在しないからです。第二引数の `arguments` には query に渡した引数が渡されます。例えば本チュートリアルでは launch query を使用する際に id を引数として渡していますので、この値を参照することができます。第三引数は `context` を受け取りますが、これを分解してデータソースを参照しています。これを通して resolver からデータソースにアクセスすることが可能です。

> The code above shows the resolver functions for the `Query` type fields: `launches`, `launch`, and `me`. The first argument to our _top-level_ resolvers, `parent`, is always blank because it refers to the root of our graph. The second argument refers to any `arguments` passed into our query, which we use in our `launch` query to fetch a launch by its id. Finally, we destructure our data sources from the third argument, `context`, in order to call them in our resolvers.

この resolver は `LaunchAPI` と `UserAPI` data sources にロジックを閉じ込めたことによって、とてもシンプルかつ簡潔なものになっています。このように resolver はなるべくロジックを書かない薄いものにするのがベストプラクティスであり、こうすることでリファクタをする際に API を壊す危険性がなくなります。

> Our resolvers are simple and concise because the logic is embedded in the `LaunchAPI` and `UserAPI` data sources. We recommend keeping your resolvers thin as a best practice, which allows you to safely refactor without worrying about breaking your API.

### Run queries in the playground

Apollo server には最初から GraphQL Playground が組み込まれているので、何も準備をしなくても query を実行したり schema がどうなっているのかを確認することができます。`npm start` を実行してサーバーを立ち上げブラウザから `http://localhost:4000/` にアクセスし GraphQL Playground を表示しましょう。

> Apollo Server sets up GraphQL Playground so that you can run queries and explore your schema with ease. Go ahead and start your server by running `npm start` and open up the playground in a browser window at `http://localhost:4000/`.


以下の GraphQL query を複写して playground の左側に貼り付けます。その後 play ボタンをクリックしてレスポンスを受け取りましょう。

> Start by copying the GraphQL query below and pasting it in the left side of the playground. Then, hit the play button at the center to get a response.

```graphql
query GetLaunches {
  launches {
    id
    mission {
      name
    }
  }
}
```

GraphQL の query を書く場合には、**operation keyword**(query とか mutation) からはじめその次にその名前もつけるようにしましょう。（上記の例では `GetLaunches` というのが名前に該当します。）このように識別用の名前をつけることで、Apollo developer tool からみた際に見つけやすくなるので、とても重要です。Query name（例えば launches）の後ろにはペアのカーリーブレイス（`{}`）を配置します。これによって内部が query の body であることを示します。ここまでをまとめると上記の query は Query type の `launches` filed にたいするオペレーションであり、`{}` の内部が body であります。さてこの内部で **selection set** を定義します。**selection set** とは query のレスポンスとしてどの field が欲しいのかということを示す部分です。

> When you write a GraphQL query, you always want to start with the **operation keyword** (either query or mutation) and its name (like `GetLaunches`). It's important to give your queries descriptive names so they're discoverable in Apollo developer tooling. Next, we use a pair of curly braces after the query name to indicate the body of our query. We specify the `launches` field on the `Query` type and use another pair of curly braces to indicate a **selection set**. The selection set describes which fields we want our query response to contain.

GraphQL がすごいのは、query の形がそのままレスポンスの形になる点です。例えば query に特定の field を追加したり削除したりすると、レスポンスにもそれが反映されます。

> What's awesome about GraphQL is that the shape of your query will match the shape of your response. Try adding and removing fields from your query and notice how the response shape changes.

では次に引数を受け取る query を書いていきましょう。以下のコードを playground に複写してください。そして play ボタンを押してレスポンスを受け取りましょう。

> Now, let's write a launch query that accepts an argument. Copy the query below and paste it in the playground. Then, click the play button to get a response.

```graphql
query GetLaunchById {
  launch(id: 60) {
    id
    rocket {
      id
      type
    }
  }
}
```

引数の `60` をハードコーディングするのではなく playground の左下にある query variables タブから引数を与えられるようにしましょう。まずは query を以下に変更します。

> Instead of hard coding the argument `60`, you can also set variables in the bottom left corner. Here's how to run that same query with variables:

```graphql
query GetLaunchById($id: ID!) {
  launch(id: $id) {
    id
    rocket {
      id
      type
    }
  }
}
```

この query に引数を与えるためには、Query Variables セクションを開いて `{ "id": 60 }` を追加します。こうすることで `GetLaunchById` が `$id` として `60` を受け取り、これを `launch(id: $id)` として使用します。

> You can paste `{ "id": 60 }` into the Query Variables section below before running your query. Feel free to experiment with running more queries before moving on to the next section.

### Paginated queries

`launches` query を実行すると大量の発射予定データが返ってきてしまいます。その結果アプリケーションが遅くなる可能性があります。一度に大量のデータを取得しないようにするためにはどうしたらいいでしょうか？

> Running the `launches` query returned a large data set of launches, which can slow down our app. How can we ensure we're not fetching too much data at once?

**Pagination** が解決策の一つです。**Pagination** 機能を使うことで、サーバーからデータを小さな単位で返すことができます。複数ページコンテンツを扱うにあたっては Cursor-based pagination を使うのが推奨の方法です。そうすることでアイテムを飛ばしてしまったり、同じアイテムが重複して表示されてしまうバグを防ぐことができます。Cursor-based pagination は constant pointer (もしくは **cursor**) を使って、次はどのアイテムからフェッチするのが正しいのかというのを特定します。

> **Pagination** is a solution to this problem that ensures that the server only sends data in small chunks. Cursor-based pagination is our recommended approach over numbered pages, because it eliminates the possibility of skipping items and displaying the same item more than once. In cursor-based pagination, a constant pointer (or **cursor**) is used to keep track of where in the data set the next items should be fetched from.

では Graph API で cursor-based pagination を使ってみましょう。`src/schema.js` を開いて `Query` type の `launches` フィールドを次のように更新しましょう。また `LaunchConnection` という新たな type を追加します。

> We'll use cursor-based pagination for our graph API. Open up the `src/schema.js` file and update the `Query` type with `launches` and also add a new type called `LaunchConnection` to the schema as shown below:

_src/schema.js_

```graphql
type Query {
  launches( # replace the current launches query with this one.
    """
    The number of results to show. Must be >= 1. Default = 20
    """
    pageSize: Int
    """
    If you add a cursor here, it will only return results _after_ this cursor
    """
    after: String
  ): LaunchConnection!
  launch(id: ID!): Launch
  me: User
}

"""
Simple wrapper around our list of launches that contains a cursor to the
last item in the list. Pass this cursor to the launches query to fetch results
after these.
"""
type LaunchConnection { # add this below the Query type as an additional type.
  cursor: String!
  hasMore: Boolean!
  launches: [Launch]!
}
...
```

コメントが追加されているのに気づかれましたでしょうか。（docstrings とも呼ばれます）スキーマ内で `"""` となっている部分です。このコメントを追加することで playground の docs からこの type をみた際に、追加の情報として表示されます。さて `launches` query は二つの引数 `pageSize` と `after` を受け取って、`LaunchConnection` を返します。`LaunchConnection` type は発射予定のリストに加えて、`cursor` フィールドを返します。`cursor` フィールドは、リストのどの位置に今いるのかという値です。また`hasMore` も返しており、これはまだフェッチできるデータがあるかどうかということを示す値です。

> You'll also notice we've added comments (also called docstrings) to our schema, indicated by `"""`. Now, the `launches` query takes in two parameters, `pageSize` and `after`, and returns a `LaunchConnection`. The `LaunchConnection` type returns a result that shows the list of launches, in addition to a `cursor` field that keeps track of where we are in the list and a `hasMore` field to indicate if there's more data to be fetched.

完成版の `src/utils.js` には `paginateResults` という関数が実装されています。この関数はサーバーから取得したデータを paginating するための機能を提供します。ではこれを使って resolver 関数を修正しましょう。

Open up the `src/utils.js` file in the repo you cloned in the previous section and check out the `paginateResults` function. The `paginateResults` function in the file is a helper function for paginating data from the server. Now, let's update the necessary resolver functions to accommodate pagination.

まず `paginateResults` 関数を `src/resolvers.js` で読み込んで `launches` を以下のコードに置き換えます。

> Let's import `paginateResults` and replace the `launches` resolver function in the `src/resolvers.js` file with the code below:

_src/resolvers.js_

```js{1,5-26}
const { paginateResults } = require('./utils');

module.exports = {
  Query: {
    launches: async (_, { pageSize = 20, after }, { dataSources }) => {
      const allLaunches = await dataSources.launchAPI.getAllLaunches();
      // we want these in reverse chronological order
      allLaunches.reverse();

      const launches = paginateResults({
        after,
        pageSize,
        results: allLaunches
      });

      return {
        launches,
        cursor: launches.length ? launches[launches.length - 1].cursor : null,
        // if the cursor of the end of the paginated results is the same as the
        // last item in _all_ results, then there are no more results after this
        hasMore: launches.length
          ? launches[launches.length - 1].cursor !==
            allLaunches[allLaunches.length - 1].cursor
          : false
      };
    },
    launch: (_, { id }, { dataSources }) =>
      dataSources.launchAPI.getLaunchById({ launchId: id }),
     me: async (_, __, { dataSources }) =>
      dataSources.userAPI.findOrCreateUser(),
  }
};
```

では今実装した cursor-based pagination が正常に機能するかテストしましょう。もし　graph API サーバーが止まっているなら `npm start` で再起動させ、それから playground で以下の query を実行します。

> Let's test the cursor-based pagination we just implemented. If you stopped your server, go ahead and restart your graph API again with `npm start`, and run this query in the c:

```graphql
query GetLaunches {
  launches(pageSize: 3) {
    launches {
      id
      mission {
        name
      }
    }
  }
}
```

この pagination 機能のおかげで、全ての発射予定ではなく、三つの予定だけが取得できるようになりました。

> Thanks to our pagination implementation, you should only see three launches returned back from our API.

## Write resolvers on types

Resolver は、schema のどんな type に対応するものでも書くことができます。つまり query や mutation だけにしか resolver を書くことができないわけではないのです。この性質のおかげで GraphQL は非常に柔軟性が高いものとなっています。

> It's important to note that you can write resolvers for any types in your schema, not just queries and mutations. This is what makes GraphQL so flexible.

気づかれたかもしれませんが、全ての type に対して resolver を書いてはいません。それでも query は正常に動いています。（訳注：例えば Rocket type に対応する resolver は書かれていないが、正常に id, name ,type といったフィールドが展開されている。）GraphQL では resolver が書かれていない場合には default resolver が動作します。親オブジェクトが必要なプロパティを持っていれば、この default resolver によってオブジェクトが展開されるので、わざわざ書く必要はありません。

> You may have noticed that we haven't written resolvers for all our types, yet our queries still run successfully. GraphQL has default resolvers; therefore, we don't have to write a resolver for a field if the parent object has a property with the same name.

ですが、resolver を type に対して書く必要のあるケースをみてみましょう。`Mission` type です。`src/resolvers.js` に移動して以下のコードを `Query` の下に配置しましょう。

> Let's look at a case where we do want to write a resolver on our `Mission` type. Navigate to `src/resolvers.js` and copy this resolver into our resolver map underneath the `Query` property:

_src/resolvers.js_

```js
Mission: {
  // make sure the default size is 'large' in case user doesn't specify
  missionPatch: (mission, { size } = { size: 'LARGE' }) => {
    return size === 'SMALL'
      ? mission.missionPatchSmall
      : mission.missionPatchLarge;
  },
},
```

_src/schema.js_
```js
  type Mission {
    # ... with rest of schema
    missionPatch(mission: String, size: PatchSize): String
  }
```

最初の引数は、親オブジェクトを受け取ります。この場合ではあれば mission object ですね。二つ目の引数はこの field に対して渡された引数ですが、これには `size` が含まれており、この値によって `mission` オブジェクトのどのプロパティを返すかを決定しています。

> The first argument passed into our resolver is the parent, which refers to the mission object. The second argument is the size we pass to our `missionPatch` field, which we use to determine which property on the mission object we want our field to resolve to.

`Query` 以外の type に対応する resolver の書き方がわかったところで、`Launch` と `User` にも resolver を与えましょう。以下のコードを複写して対応する resolver を変更します。

> Now that we know how to add resolvers on types other than `Query` and `Mission`, let's add some more resolvers to the `Launch` and `User` types. Copy this code into your resolver map:

_src/resolvers.js_

```js
Launch: {
  isBooked: async (launch, _, { dataSources }) =>
    dataSources.userAPI.isBookedOnLaunch({ launchId: launch.id }),
},
User: {
  trips: async (_, __, { dataSources }) => {
    // get ids of launches by user
    const launchIds = await dataSources.userAPI.getLaunchIdsByUser();

    if (!launchIds.length) return [];

    // look up those launches by their ids
    return (
      dataSources.launchAPI.getLaunchesByIds({
        launchIds,
      }) || []
    );
  },
},
```

上のコードをみて疑問が浮かんだかもしれません。予約した情報を取得する際に必要なユーザー情報はどうやって取得するんだろうと。いい観点です。まだ実装していませんが、ユーザー認証の機能が必要です。ここからはユーザー認証の実装とユーザー情報を context に含ませる方法について取り上げます。そのあと `Mutation` resolver を紹介します。

> You may be wondering where we're getting the user from in order to fetch their booked launches. This is a great observation - we still need to authenticate our user! Let's learn how to authenticate users and attach their user information to the context in the next section before we move onto `Mutation` resolvers.

## Authenticate users

Access control 機能はほとんど全てのアプリケーションに必要なものをといっていいでしょう。このチュートリアルではユーザー認証の根幹的な概念を伝えることに集中し、具体的な実装については取り上げません。

> Access control is a feature that almost every app will have to handle at some point. In this tutorial, we're going to focus on teaching you the essential concepts of authenticating users instead of focusing on a specific implementation.

概要は以下です。

> Here are the steps you'll want to follow:

1. `ApolloServer` instance に与える context 関数は、GraphQL operation が実行され API が叩かれるたびに呼び出されます。その際にこの関数は request object を受け取ります。この request object から the authorization headers 情報を読み取ります。
1. ユーザーの認証は context 関数内で行います。
1. ユーザー認証が完了したら、そのユーザーを context 関数が返すオブジェクトに紐づけます。これによってユーザー情報を data source や resolver から読み取ることができるようになります。これを用いてデータへのアクセスを許可するかどうかを判断すればいいわけです。

> 1. The context function on your `ApolloServer` instance is called with the request object each time a GraphQL operation hits your API. Use this request object to read the authorization headers.
> 1. Authenticate the user within the context function.
> 1. Once the user is authenticated, attach the user to the object returned from the context function. This allows us to read the user's information from within our data sources and resolvers, so we can authorize whether they can access the data.

では `src/index.js` を開いて `context` 関数を更新しましょう。

> Let's open up `src/index.js` and update the `context` function on `ApolloServer` to the code shown below:

_src/index.js_

```js{1,4,8,10}
const isEmail = require('isemail');

const server = new ApolloServer({
  context: async ({ req }) => {
    // simple auth check on every request
    const auth = req.headers && req.headers.authorization || '';
    const email = Buffer.from(auth, 'base64').toString('ascii');

    if (!isEmail.validate(email)) return { user: null };

    // find a user by their email
    const users = await store.users.findOrCreate({ where: { email } });
    const user = users && users[0] || null;

    return { user: { ...user.dataValues } };
  },
  // .... with the rest of the server object code below, typeDefs, resolvers, etc....
```

Request に含まれる Authorization header をチェックし、その情報を元にデーターベースからユーザーが存在するかを探します。そして存在すればそのユーザーを `context` に付与します。この実装は簡易的なものなので全く安全でないので、この実装を本当のプロダクトで使ったほうがいいというわけでは決してありませんが、しかし概念としては本物のアプリケーションにおける認証においても通じるものです。（訳注：メールアドレスだけで単に DB から照合してしまうと、当然だが嘘のメールアドレスを送られても通ってしまう。そこに問題がある。しかし request から値を読み出し、それを用いて認証を行い、通った場合には context に対してユーザー情報等を含ませる、というフロー自体はプロダクトでも同様であるということ。）

> Just like in the steps outlined above, we're checking the authorization headers on the request, authenticating the user by looking up their credentials in the database, and attaching the user to the `context`. While we definitely don't advocate using this specific implementation in production since it's not secure, all of the concepts outlined here are transferable to how you'll implement authentication in a real world application.

`authorization` headers に渡される token はどうやって作るのでしょうか。次のセクションではその疑問に応えるために `login` mutation に対する resolver を書いていくことにしましょう。

> How do we create the token passed to the `authorization` headers? Let's move on to the next section, so we can write our resolver for the `login` mutation.

## Write Mutation resolvers

`Mutation` の resolver も、今まで書いてきた resolver とほとんど同じです。まず `login` resolver を作成し、認証フローを完成させましょう。以下のコードを `Query` resolver 配下に追加しましょう。

> Writing `Mutation` resolvers is similar to the resolvers we've already written. First, let's write the `login` resolver to complete our authentication flow. Add the code below to your resolver map underneath the `Query` resolvers:

_src/resolvers.js_

```js
Mutation: {
  login: async (_, { email }, { dataSources }) => {
    const user = await dataSources.userAPI.findOrCreateUser({ email });
    if (user) return Buffer.from(email).toString('base64');
  }
},
```

`login` resolver は email adress を受け取って token を返す関数ですが、ユーザーが 存在する時のみ token を返します。後ほどこの token を client 側で保持する方法も学習します。

> The `login` resolver receives an email address and returns a token if a user exists. In a later section, we'll learn how to save that token on the client.

さらに `bookTrips` と `cancelTrip` の `Mutation` も resolver に追加しましょう。

> Now, let's add the resolvers for `bookTrips` and `cancelTrip` to `Mutation`:

_src/resolvers.js_

```js
Mutation: {
  bookTrips: async (_, { launchIds }, { dataSources }) => {
    const results = await dataSources.userAPI.bookTrips({ launchIds });
    const launches = await dataSources.launchAPI.getLaunchesByIds({
      launchIds,
    });

    return {
      success: results && results.length === launchIds.length,
      message:
        results.length === launchIds.length
          ? 'trips booked successfully'
          : `the following launches couldn't be booked: ${launchIds.filter(
              id => !results.includes(id),
            )}`,
      launches,
    };
  },
  cancelTrip: async (_, { launchId }, { dataSources }) => {
    const result = await dataSources.userAPI.cancelTrip({ launchId });

    if (!result)
      return {
        success: false,
        message: 'failed to cancel trip',
      };

    const launch = await dataSources.launchAPI.getLaunchById({ launchId });
    return {
      success: true,
      message: 'trip cancelled',
      launches: [launch],
    };
  },
},
```

`bookTrips` と `cancelTrip` のどちらも、schema で定義した `TripUpdateResponse` type の値を返す必要があります。これには「処理が成功したかどうか」、「状態メッセージ」、「予約もしくはキャンセルした発射予定アイテムに関する情報の配列」が含まれます。`bookTrips` mutation は少々複雑な状況が発生しがちです。つまり、予約もしくはキャンセル処理の一部は成功して、一部は失敗するという場合があるからです。いまのところは実装を簡単なものにするため、部分的に成功した場合には `message` でそれを示すようにだけすることにします。

> Both `bookTrips` and `cancelTrip` must return the properties specified on our `TripUpdateResponse` type from our schema, which contains a success indicator, a status message, and an array of launches that we've either booked or cancelled. The `bookTrips` mutation can get tricky because we have to account for a partial success where some launches could be booked and some could fail. Right now, we're simply indicating a partial success in the `message` field to keep it simple.

### Run mutations in the playground

さあ面白いところです。mutation を playground で実行しましょう！ブラウザで playground にアクセスし、リロードし、operation を発行しましょう。

It's time for the fun part - running our mutations in the playground! Go back to the playground in your browser and reload the schema with the little return arrow at the top on the right of the address line.

GraphQL mutation は query と完全に同じ構造ですが、`query` ではなく `mutation` というキーワードを使用します。以下の mutation を playground に複写し、実行しましょう。

> GraphQL mutations are structured exactly like queries, except they use the `mutation` keyword. Let's copy the mutation below and run in the playground:

```graphql
mutation LoginUser {
  login(email: "daisy@apollographql.com")
}
```

すると次のような文字列が返ってくるはずです。

`ZGFpc3lAYXBvbGxvZ3JhcGhxbC5jb20=`

この文字列をコピーして次のミューテーションで使用します。

> You should receive back a string that looks like this: `ZGFpc3lAYXBvbGxvZ3JhcGhxbC5jb20=`. Copy that string because we will need it for the next mutation.

では先ほど取得した文字列を使って旅行を予約しましょう。認証されたユーザーだけが予約を実行することができます。Playground には authorization header を添付する場所があるのでこれを使って、先ほどの mutation に渡すことでユーザーの認証を完了させます。まずは以下の mutation を playground に複写します。

> Now, let's try booking some trips. Only authorized users are permitted to book trips, however. Luckily, the playground has a section where we can paste in our authorization header from the previous mutation to authenticate us as a user. First, paste this mutation into the playground:

```graphql
mutation BookTrips {
  bookTrips(launchIds: [67, 68, 69]) {
    success
    message
    launches {
      id
    }
  }
}
```

次に画面の下の方にある HTTP Headers box に authorization header を貼り付けます。

> Next, paste our authorization header into the HTTP Headers box at the bottom:

```json
{
  "authorization": "ZGFpc3lAYXBvbGxvZ3JhcGhxbC5jb20="
}
```

そうしたら mutaion を実行しましょう。すると成功メッセージとともに予約できた旅行の id が配列で返ってきます。このうように mutation のテストを playground から手動で実行してもいいのですが、実際のアプリケーションにおいてはテストを自動化し、リファクタリングが安全にできるような環境を構築したほうがいいでしょう。次のセクションでは graph のテストではなく、実際の環境で graph を起動する方法を学習します。

> Then, run the mutation. You should see a success message, along with the ids of the mutations we just booked. Testing mutations manually in the playground is a good way to explore our API, but in a real-world application, we should run automated tests so we can safely refactor our code. In the next section, you'll actually learn about running your graph in production instead of testing your graph.
