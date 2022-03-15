# 第三章

- withAuth（認證用）HOC
  - getInitialProps 函式
  - withAuth HOC 的參數
  - 測試 withAuth
  - 登入頁面（Login）與 NProgress
  - 非同步（asynchronous）執行及 callback
    - Promise.then
    - async/await
  - 整合 Google OAuth API
    - setupGoogle 函式、verify 函式、passport 及 strategy
    - /auth/google、/oauth2callback 及 /logout Express 路由
    - User.publicFields 及 User.signInOrSignUp 函式
    - generateSlug 函式
    - GCP（Google Cloud Platform）設定與測試

---

在我們開工之前，先取得`3-begin`的程式碼。[3-begin](https://github.com/menon-pheno/fullstack-bookstore/tree/master/book/3-begin)資料夾位於[fullstack-bookstore repo](https://github.com/menon-pheno/fullstack-bookstore)`book`的目錄內。

- 如果你還沒有將 fullstack-bookstore 給 clone 下來的話，用`git clone https://github.com/menon-pheno/fullstack-bookstore`這個指令將 repo 複製到你的電腦上
- 注意：如果你想要用自己的 GitHub 帳號自己管理程式的話，你應該將我們的 repo fork 出來並且執行`git clone https://github.com/<你的 github 名稱>/fullstack-bookstore.git`。這樣你就可以將你的改動直接 push 到你的`fullstack-bookstore` repo
- 在`3-begin`的資料夾內執行`yarn`來安裝所有的套件

我們在第三章有新增幾個套件：

- `"lodash"`
- `"nprogress"`
- `"passport"`
- `"passport-google-oauth"`

看一下第三章的 [package.json](https://github.com/menon-pheno/fullstack-bookstore/blob/master/book/3-begin/package.json)。

請確定使用我們指定的套件跟版本，並忽略任何升級的警告。我們會定期更新套件且測試相容性。我們無法確保新的套件版本與目前的程式碼都相容，套件升級時有的時候會導致一些預料之外的問題。

記得將你第二章建立的 `.env` 檔案放到專案的根目錄下。到本章的尾聲時，你會另外加上 `GOOGLE_CLIENTID` 以及 `GOOGLE_CLIENTSECRET` 的環境變數到 `.env` 檔案內。

我們鼓勵且歡迎你在閱讀本章的時候，可以在我們的 GitHub repo: [https://github.com/menon-pheno/fullstack-bookstore/issues/new](https://github.com/menon-pheno/fullstack-bookstore)回報任何 bug、錯字或是任何解釋不清楚的地方。

---

在本章，我們會建立一個名為 `withAuth` 的高階元件（higher-order component, HOC）。這個 HOC 與我們的 `App` HOC 類似，會將我們的頁面給包覆起來。不同的點在於，`App` 預設上會將所有的頁面都包覆，而 `withAuth` 則是我們在 export 頁面時會特別指定是否要使用這個 HOC 來包覆該頁面。`App` 的主要責任是提供我們程式個頁面所共用的版面（layout） - 例如 Material-UI 的主題（theme）、`Header` 元件以及我們會加上的 `Notifier` 元件。而我們 `withAuth` 元件的主要責任則是檢查使用者是否有登入，並在有登入的時候將 `user` 以 prop 的方式傳入頁面元件內。

在 `withAuth` HOC 內，我們會使用 Next.js 的 `getInitialProps` 函式來取得 `user` prop 該有的值，以傳入到頁面元件內。我們在本章中也會稍微離題以更深入的探討一下 `getInitialProps` 這個 Next.js 所提供的函式。

總結一下本章會涉獵的內容，除了實作 `withAuth` HOC 以外，我們會：

- 探討 `getInitialProps` 函式
- 建立我們的 `Login` 頁面，並引用 `Nprogress` 進度條
- 學習 `Promise.then`、`async/await` 的概念
- 整合 Google OAuth API

---

## withAuth（認證用）HOC

要為應用程式導入使用者認證的功能不算是個小工程。因此我們花一整章的時間來做這件事。我們首要的工作是探討並定義實作一個會將 `user` 以 prop 的方式傳到所包覆的頁面元件的 `withAuth` HOC。接著我們會稍微離題探討非同步執行程式這個主題，然後將我們的網頁程式與 Google OAuth API 整合，最後再來測試我們整個使用者驗證的流程。

本節的兩個主要目標為：

- 在 `lib/withAuth.jsx` 定義 `withAuth` HOC
- 用一個手動於 MongoDB 資料庫建立的 `User` 文件來測試我們的 `withAuth` HOC

首先，我們接下來會藉由執行 `yarn dev` 而非 `yarn dev-express` 來啟動我們的伺服器。在 `package.json` 的 `scripts` 屬性下將：

```JSON
"dev": "next",
"dev-express": "nodemon server/server.js --watch server",
```

改成：

```JSON
"dev": "nodemon server/server.js --watch server",
```

上一章的時候，我們透過 Next.js 的 `getInitialProps` 來將使用者的資訊以 prop 的方式傳入 `Index` 頁面內。如果只有這個頁面需要這個資訊，這樣做沒什麼問題。不過，我們的網頁應用程式會有多個頁面需要這個 `user` prop 來正確地顯示資訊。如果每個頁面都這樣做，那容易出錯又不好維護。

為了提升程式碼重用性，我們可以透過建立一個 HOC 來將需要使用 `user` prop 的頁面給包覆起來，並由這個 HOC 負責將 `user` prop 給傳入所包覆的頁面。所以我們建立一個 `withAuth` 並使用 `withAuth.getInitialProps` 來包覆 `Index` 及其他需要 `user` 資訊的頁面來取代我們目前直接在 `Index` 頁面使用 `Index.getInitialProps` 的方式。

我們將 `withAuth` HOC 定義在 `lib` 目錄內的 `lib/withAuth.jsx`。在 `withAuth` 元件內我們會指定幾個簡單的 boolean 參數 - 例如 `loginRequired` 來決定所包覆的頁面是否一定要通過認證（`user` 必須不是 `null` 或 `undefined`）。接這我們就是把頁面用 `withAuth` 給包起來，並且透過這些 boolean 參數來定義所包覆的頁面該有的行為。下面是一個帶入 `loginRequired: true` 的例子：

```JavaScript
export default withAuth(Index, { loginRequired: true });
```

上面這行代表 `Index` 這個頁面需要一個有登入的使用者才能造訪。

所有針對管理者（Admin）以及顧客（Customer）角色所設計的頁面都需要是一個有登入的使用者才能造訪；因此這些頁面都應該用 `loginRequired: true` 的 `withAuth` HOC 給包覆起來。在本章中，你可以將被 `withAuth` 包覆的 `Index` 頁面想成是一個使用者的專屬儀表板（dashboard）- 當使用者成功登入後被轉導到的首頁。

我們再看一次 `Index` 頁面的 export 型態：

```JavaScript
export default withAuth(Index, { loginRequired: true });
```

可以看到 `withAuth` HOC 將 `Index` 包覆起來，並且 `loginRequired` 則指定使用者必須有成功登入才能造訪此頁面。

因此，`withAuth` 這個函式會需要：

1. 頁面元件（`BaseComponent`）
2. 具有 `loginRequired` 跟 `logoutRequired` 等屬性的物件

這兩個參數。而最後 `withAuth` 會回傳一個 `App` 元件，其內容實際上是：

```JSX
<BaseComponent {...this.props} />
```

建立 `lib/withAuth.jsx` 並且先將以下的實作給填上：

```JSX
import React from 'react';
import PropTypes from 'prop-types';
import Router from 'next/router';

const globalUser = null;

export default function withAuth(
  BaseComponent,
  { loginRequired = true, logoutRequired = false } = {},
) {
  const propTypes = {
    user: PropTypes.shape({
      id: PropTypes.string,
      isAdmin: PropTypes.bool,
    }),
    isFromServer: PropTypes.bool.isRequired,
  };

  const defaultProps = {
    user: null,
  };

  class App extends React.Component {
    static async getInitialProps(ctx) {
      // 1. getInitialProps
    }

    componentDidMount() {
      // 2. componentDidMount
    }

    render() {
      // 3. render

      return (
        <>
          <BaseComponent {...this.props} />
        </>
      );
    }
  }

  App.propTypes = propTypes;
  App.defaultProps = defaultProps;

  return App;
}
```

你可能會好奇，為何 `withAuth` 是一個回傳元件的函式，但是 `pages/` 內的 `MyApp` 及 `MyDocument` HOC 卻是單純的元件。那是因為 `App` 及 `Document` 嚴格來說並不是 HOC（我們是為了不要將事情複雜化而單純叫它們 HOC） - 實際上它們是 Next.js 內建的 `App` 及 `Document` HOC 的延伸（extend）。我們定義的 `MyApp` 及 `MyDocument` 對應的各自延伸 Next.js 的 `App` 及 `Doucment` HOC。所以嚴格來說 `MyApp` 及 `MyDocument` 是 HOC 的延伸，而非 HOC。反觀我們的 `withAuth` 則是一個獨立的 HOC，可以參考 HOC 的定義：

[https://blog.jakoblind.no/simple-explanation-of-higher-order-components-hoc/#the-most-simple-hoc](https://blog.jakoblind.no/simple-explanation-of-higher-order-components-hoc/#the-most-simple-hoc)

```JSX
// 將一個元件(WrappedComponent)以參數傳入
function simpleHOC(WrappedComponent) {
    // 回傳一個新的 anonymous 元件
    return class extends React.Component {
        render() {
            return <WrappedComponent {...this.props} />;
        }
    }
}
```

我們定義 `withAuth` 的方式就跟上面是一致的。唯一的差異是我們透過 `class App extends` 然後再回傳 `return App` 而上放的範例使用 `return class extends`，這只是單純語法的不同。值得一提的是，Next.js **內部**也是如此定義 `App` 及 `Document` HOC 的。

我們在第一章就談過指定 prop 的型態以及預設值，所以在這裡就不贅述。

另外一個值得注意的是 - `<>` 是 `React.Fragment` 的簡化語法：

[https://reactjs.org/docs/fragments.html#short-syntax](https://reactjs.org/docs/fragments.html#short-syntax)

在接下來的幾個小節，我們會定義下列的函式，並且測試我們包覆了 `Index` 頁面的 `withAuth` HOC：

- `withAuth.getInitialProps`
- `withAuth.componentDidMount`
- `withAuth.render`

---

### getInitialProps 函式

我們在本節定義 `withAuth.getInitialProps` 函式。我們之前已經有使用過 `getInitialProps` 這個函式（`Index` 頁面）。Next.js 使用 [getInitialProps](https://github.com/zeit/next.js#fetching-data-and-component-lifecylce) 函式來取得頁面元件所需要的 prop 資料。HOC 及 Next.js 頁面（元件）兩者都可以使用這個函式，但是其子元件（childe component）是不允許使用此函式。子元件必須透過父元件傳入 prop 來得到資料。

`getInitialProps` 函式是設計為可以在瀏覽器以及伺服器端執行的，因此當你知道你的頁面同時有可能是伺服器渲染及瀏覽器渲染的時候，這個函式就很適用。若你的頁面只會由瀏覽器渲染，你可以透過 `componentDidMount`（或是用 hook）以取得需要的 prop 資料。

我們的 `withAuth.getInitialProps` 需要計算出 `user` 以及 `isFromServer` 的值，然後將這兩個值以 props 的方式傳到 `App` 元件內。我們的 `withAuth.getInitialProps` 函式定義如下：

```JavaScript
static async getInitialProps(ctx) {
    const isFromServer = typeof window === 'undefined';
    const user = ctx.req ? ctx.req.user && ctx.req.user.toObject() : globalUser;

    if (isFromServer && user) {
    user._id = user._id.toString();
    }

    const props = { user, isFromServer };

    if (BaseComponent.getInitialProps) {
    Object.assign(props, (await BaseComponent.getInitialProps(ctx)) || {});
    }

    return props;
}
```

`isFromServer` 被定義為 `typeof window === 'undefined'`。換言之，當頁面是伺服器渲染的時候，`typeof window === 'undefined'` 會是 `true`，也因此 `isFromServer` 會是 `true`。而我們使用 `isFromServer` 的方式如下（待會 `withAuth.componentDidMount` 也會使用到）：

```JavaScript
if (isFromServer && user) {
    user._id = user._id.toString();
}
```

這裡注意到，在伺服器端處理 `user._id` 需要多呼叫一個 `toString`，可以參考以下兩個連結：

[https://docs.mongodb.com/manual/reference/method/ObjectId.toString/](https://docs.mongodb.com/manual/reference/method/ObjectId.toString/)

[https://stackoverflow.com/questions/13104690/nodejs-mongodb-object-id-to-string](https://stackoverflow.com/questions/13104690/nodejs-mongodb-object-id-to-string)

我們來用個 `console.log()` 來驗證一下從伺服器端來的 `user._id` 並非 `string` 而是 `object`（因此需要呼叫 `toString`），不過由於我們還未將 `Index` 頁面與 `withAuth` 結合，因此我們透過 `Index.getInitialProps` 來示範，將 `Index.getInitialProps` 改成以下：

```JavaScript
Index.getInitialProps = async (ctx) => {
  const isFromServer = typeof window === 'undefined';
  const user = ctx.query ? ctx.query.user && ctx.query.user.toObject() : null;

  if (isFromServer && user) {
    console.log('Index page server getInitialProps...');
    console.log('before', typeof user._id, user._id);
    user._id = user._id.toString();
    console.log('after', typeof user._id, user._id);
  }

  return {
    user: ctx.query.user,
  };
};
```

重整你的 `http://localhost:8000` 頁面，讓它取得一個伺服器渲染的頁面，你的終端機會 log 出：

```
Index page server getInitialProps...
before object 620493d3fd9c514ea9e29eaf
after string 620493d3fd9c514ea9e29eaf
```

可以看到 `user._id` 本來是一個 `object` 的型態，因此需要呼叫 `toString` 將之轉型為 `string`。當然你看到的 `user._id` 會與我們這邊有所不同。

我們傳入了 `{loginRequired = true, logoutRequired = false}` 這個參數到我們的 `withAuth` HOC，不過我們目前 `withAuth` 的邏輯都還沒有用到這個參數。我們在下個小節即將在定義 `withAuth.componentDidMount` 以及 `withAuth.render` 時使用它們。

---

### withAuth HOC 的參數

我們在這邊討論一下 `withAuth` HOC 的 `loginRequired` 以及 `logoutRequired` 參數的作用為何。我們在第六章會另外再多加一個 `adminRequired` 參數。

我們的網頁應用程式應該如何處理已登出的使用者呢？一個好的使用體驗（user experience, UX）應該是在當已登出的使用者想要造訪一個需要登入才能使用的頁面時，將使用者導到 `Login` 頁面。換個角度來看，我們透過 `withAuth` 來指定 `Index` 頁面只能由已登入的使用者使用，而 `Login` 頁面只能由未登入的使用者使用。

因此，我們可以在 `Index` 頁面使用 `loginRequired` 為 `true` 如下：

```JavaScript
export default withAuth(Index, { loginRequired: true });
```

而 `Login` 頁面則可以將 `logoutRequired` 設為 `true` 如下：

```JavaScript
export default withAuth(Login, { logoutRequired: true });
```

我們 `lib/withAuth.jsx` 定義上面的參數如下：

```JavaScript
{ loginRequired = true, logoutRequired = false } = {}
```

這個代表說，假設我們沒有指定 `loginRequired` 的值，它預設就會是 `loginRequired: true`。因此我們只要簡單的如下調整 `Index`：

```JavaScript
export default withAuth(Index);
```

這樣就會指定 `Index` 為只能由已登入的使用者造訪。

但是我們的 `Login` 頁面仍然需要將 `logoutRequired: true` 傳入（因為 `logoutRequired` 的預設值是 `false`）。所以我們 export `Login` 頁面元件的方式依然維持為：

```JavaScript
export default withAuth(Login, { logoutRequired: true });
```

到這裡你應該了解我們為何需要 `loginRequired` 及 `logoutRequired` 這兩個參數，以及 `withAuth` 包覆頁面時何時需要傳入指定值。我們接下來就是要利用這些參數在 `withAuth` 內設定對應的邏輯。

我們將要在 `withAuth.componentDidMount` 內實作轉導的相關邏輯。

想像一下使用者造訪由 `withAuth` 包覆的 `Index` 頁面：

```JavaScript
export default withAuth(Index);
```

這代表 `loginRequired = true` 及 `logoutRequired = false`。如果使用者未登入，則 `!user` 為 `true`，那我們的應用程式會將使用者轉導至 `Login` 頁面。

當使用者想要造訪由 `withAuth` 包覆的 `Login` 頁面：

```JavaScript
export default withAuth(Login, { logoutRequired: true });
```

由於 `logoutRequired = true`，所以如果使用者已登入，則我們將會將使用者轉導至 `Index` 頁面。

將以上兩個情境結合起來：

```JavaScript
componentDidMount() {
    const { user } = this.props;

    if (loginRequired && !logoutRequired && !user) {
        Router.push('/login');
        return;
    }

    if (logoutRequired && user) {
        Router.push('/');
    }
}
```

另外，在我們上述的兩個情境轉導之前，我們也要禁止我們的網頁應用程式去渲染所造訪的網頁，因此：

```JavaScript
render() {
    const { user } = this.props;

    if (loginRequired && !logoutRequired && !user) {
        return null;
    }

    if (logoutRequired && user) {
        return null;
    }

    return (
        <>
            <BaseComponent {...this.props} />
        </>
    );
}
```

將以上的事情做完後，你的 `lib/withAuth.jsx` 應該如下：

```JSX
import React from 'react';
import PropTypes from 'prop-types';
import Router from 'next/router';

let globalUser = null;

export default function withAuth(
  BaseComponent,
  { loginRequired = true, logoutRequired = false } = {},
) {
  const propTypes = {
    user: PropTypes.shape({
      id: PropTypes.string,
      isAdmin: PropTypes.bool,
    }),
    isFromServer: PropTypes.bool.isRequired,
  };

  const defaultProps = {
    user: null,
  };

  class App extends React.Component {
    static async getInitialProps(ctx) {
      const isFromServer = typeof window === 'undefined';
      const user = ctx.req ? ctx.req.user && ctx.req.user.toObject() : globalUser;

      if (isFromServer && user) {
        user._id = user._id.toString();
      }

      const props = { user, isFromServer };

      if (BaseComponent.getInitialProps) {
        Object.assign(props, (await BaseComponent.getInitialProps(ctx)) || {});
      }

      return props;
    }

    componentDidMount() {
      const { user, isFromServer } = this.props;

      if (isFromServer) {
        globalUser = user;
      }

      if (loginRequired && !logoutRequired && !user) {
        Router.push('/login');
        return;
      }

      if (logoutRequired && user) {
        Router.push('/');
      }
    }

    render() {
      const { user } = this.props;

      if (loginRequired && !logoutRequired && !user) {
        return null;
      }

      if (logoutRequired && user) {
        return null;
      }

      return (
        <>
          <BaseComponent {...this.props} />
        </>
      );
    }
  }

  App.propTypes = propTypes;
  App.defaultProps = defaultProps;

  return App;
}
```

這樣我們的 `withAuth` HOC 就完成了，下個小節我們就可以來測試驗證了！

---

### 測試 withAuth

到這邊，我們有個 `withAuth` HOC 的初版了。

將 `withAuth` 引入至 `Index` 頁面，並且用前面所提到的方式將頁面用 `withAuth` 包覆起來。

我們之前實作 `Index` 頁面的方式為：

- 我們是用 stateless functional component 的方式實作（見第一章）
- 頁面是透過 `Index.getInitialProps` 從伺服器取得使用者資訊（見第二章）

我們測試之前，先做幾個改變：

- 將頁面元件改以 `class...extends` 語法定義
- 移除 `Index.getInitialProps`，因為我們將會使用 `withAuth` HOC 來取得資訊

做了這些改動後，你的 `pages/index.jsx` 應該如以下：

```JSX
import React from 'react';
import PropTypes from 'prop-types';
import Head from 'next/head';

import withAuth from '../lib/withAuth';

const propTypes = {
  user: PropTypes.shape({
    displayName: PropTypes.string,
    email: PropTypes.string.isRequired,
  }),
};

const defaultProps = {
  user: null,
};

class Index extends React.Component {
  render() {
    const { user } = this.props;
    return (
      <div style={{ padding: '10px 45px' }}>
        <Head>
          <title>首頁</title>
          <meta name="說明" content="這是首頁的說明資訊" />
        </Head>
        <p>已購買書籍清單</p>
        <p>
          Email:&nbsp;
          {user.email}
        </p>
      </div>
    );
  }
}

Index.propTypes = propTypes;
Index.defaultProps = defaultProps;

export default withAuth(Index);
```

VS code 編輯器會報一個警告：

```
Component should be written as a pure function
```

你可以在類別定義的上面加上 `// eslint-disable-next-line react/prefer-stateless-function` 來消除這個警告。由於我們這頁的行為會逐漸複雜化，以類別的方式比較好說明，我們會維持使用類別的實作方式來進行。

在我們測試 `Index` 之前，把 `withAuth` 內的以下這段給暫時註記起來：

```JavaScript
if (loginRequired && !logoutRequired && !user) {
    Router.push('/login');
    return;
}
```

因為我們不要在測試中被轉導。

現在如果 `yarn dev` 並且造訪 `http://localhost:8000` 的話，你會看到：

```
TypeError: Cannot read property 'email' of null
```

這代表了我們沒有正確的從伺服器端取得 `user` 資訊。`user` 資訊來自於我們第二章所定義的 `/` Express 路由。打開 `server/server.js` 看一下我們目前的 `/` Express 路由定義：

```JavaScript
server.get('/', async (req, res) => {
    req.session.foo = 'bar';
    const user = await User.findOne({ slug: 'pheno-author' });
    app.render(req, res, '/', { user });
});
```

由於我們是將 `user` 以 `app.render` 的第四個參數傳入，這代表的是當時我們是透過 `Index.getIntialProps` 的 `ctx.query.user` 取得使用者資訊。但是注意到我們在 `withAuth` 內定義 `user` prop 的方式為：

```JavaScript
const user = ctx.req ? ctx.req.user && ctx.req.user.toObject() : globalUser;
```

這代表我們 `/` Express 路由的邏輯需要調整。我們需要透過 `ctx.req.user` 而非 `ctx.query.user` 來取得 `user` 資訊。將 `/` Express 路由修改如下：

```JavaScript
server.get('/', async (req, res) => {
    req.session.foo = 'bar';
    const user = await User.findOne({ slug: 'pheno-author' });
    req.user = user;
    app.render(req, res, '/');
});
```

值得注意的是，我們本章晚點會整合 Google OAuth API，在整合完畢後我們要把這個 Express 路由刪除。這是因為我們用來整合 Google OAuth API 的 `passport` 套件會負責將使用者資訊提供至 `req.user`，我們就不用做這個動作了。

另外，你也需要確認你的 MongoDB 資料庫內的 `users` 集合是具有 `User` MongoDB 文件的。我們在第二章有進行過這件事情，如果你沒做這個動作，請到 MongoDB Atlas，點擊 `bookstore.users` 集合，**手動**新增以下的 `User` MongoDB 文件（可以參考第二章我們所進行的步驟）：

```JSON
createdAt: 2017-12-17T02:05:57.426+00:00
email: "pheno_the_best@yahoo.com.tw"
displayName: "黃敬強"
avatarUrl: "https://lh3.googleusercontent.com/-XdUIqdMkCWA/AAAAAAAAAAI/AAAAAAAAAAA..."
slug: "pheno-author"
```

現在 `yarn dev` 並造訪 `http://localhost:8000`：

![alt chapter 3 user](./3-images/3-chrome-user.png)

如果你在 `Index` 頁面看得到使用者的 email，那你成功的透過 `withAuth` HOC 將 `user` 以 prop 的方式傳到 `Index` 頁面了！

---

### 登入頁面（Login）與 NProgress

我們雖然在討論 `withAuth` 時有提到 `Login` 頁面，但是我們還沒有進行 `Login` 頁面的任何實作。

我們實際 export `Login` 頁面元件的時候會將 `logoutRequired` 設定為 `true`（因為 `logoutRequired` 的預設值是 `false`）：

```JavaScript
export default withAuth(Login, { logoutRequired: true });
```

我們這次將 `Login` 頁面定義為 stateless functional component，`pages/login.jsx` 如下（如果第一章與第二章有好好吸收，這個頁面理應上不難懂）：

```JSX
import Head from 'next/head';
import Button from '@material-ui/core/Button';

import withAuth from '../lib/withAuth';
import { styleLoginButton } from '../components/SharedStyles';

const Login = () => (
  <div style={{ textAlign: 'center', margin: '0 20px' }}>
    <Head>
      <title>登入 pheno 書店</title>
      <meta name="description" content="書店登入頁面" />
    </Head>
    <br />
    <p style={{ margin: '45px auto', fontSize: '44px', fontWeight: '400' }}>登入</p>
    <p>若沒有手動登出，將會維持登入 14 天</p>
    <br />
    <Button variant="contained" style={styleLoginButton} href="/auth/google">
      <img
        src="https://storage.googleapis.com/builderbook/G.svg"
        alt="使用 Google 登入"
        style={{ marginRight: '10px' }}
      />
      使用 Google 登入
    </Button>
  </div>
);

export default withAuth(Login, { logoutRequired: true });
```

我們之前的章節已經探討過 Next.js 的 `Head` 元件以及 Material-UI 的 `Button` 元件，這裡不贅述。值得注意的是 `使用 Google 登入` 這個按鈕是會將使用者轉導至 `/auth/google` 這個路由，藉由這個路由我們進行 Google OAuth API 的認證過程。本章晚點我們會定義一個 `/auth/google` Express 路由來讓伺服器知道當使用者造訪 `/auth/google` 時該進行什麼動作。我們第五章還會繼續探討 Express 路由。

`yarn dev` 並造訪 `http://localhost:8000/login`：

![alt Login](./3-images/3-chrome-login.png)

雖然我們尚未整合 Google OAuth API，但是我們的應用程式離具有使用者登入功能又再進一步了！

讓我們現在來做一個使用者體驗的改善。我們的網頁應用程式應該要有某種視覺提示來**表示**正在進行載入的動作。當頁面正在載入，或是使用者點擊了個按鈕觸發 API 時，我們應該要讓使用者知道有進度。舉例來說，當點擊我們左上角的 logo 連結時，沒有任何視覺提示讓我們知道說路由改變了，正在載入新的頁面這個行為會是使用者不預期的。對比來說，嘗試載入下面這個網址：

[http://ricostacruz.com/nprogress/](http://ricostacruz.com/nprogress/)

這個頁面在上方有個簡潔的藍色進度條。這個進度條會讓使用者知道頁面正在載入，並且給予一個大致進度狀況的呈現。

我們也在我們的網頁應用程式加上這個 `NProgress` 進度條。本章開始時的 `yarn` 已將 `nprogress` 套件給安裝了。

我們可以選擇將 `NProgress` 進度條加在我們的 `Header` 元件或是 `App` HOC。由於所有的頁面應該都用得到進度條，所以比較好的選擇會是將進度條加在 `App` 裡面，這樣所有的頁面都會具有這個元件。

打開你的 `pages/_app.jsx`，加入兩個新的引入元件 `Router` 及 `NProgress`：

```JavaScript
import CssBaseline from '@material-ui/core/CssBaseline';
import { ThemeProvider } from '@material-ui/styles';
import App from 'next/app';
import PropTypes from 'prop-types';
import React from 'react';
import Head from 'next/head';
import Router from 'next/router';
import NProgress from 'nprogress';

import { theme } from '../lib/theme';

import Header from '../components/Header';
```

接者在 import 之後在 `Router` 的幾個事件觸發時呼叫 `NProgress.start` 及 `NProgress.done`：

```JavaScript
Router.onRouteChangeStart = () => NProgress.start();
Router.onRouteChangeComplete = () => NProgress.done();
Router.onRouteChangeError = () => NProgress.done();
```

Next.js 的路由事件（router event）可以參考以下官方文件：

[https://nextjs.org/docs/api-reference/next/router#routerevents](https://nextjs.org/docs/api-reference/next/router#routerevents)

至於 `NProgress` 的 style 主要有兩種方式可以處理：

1. 到 [https://unpkg.com/nprogress@0.2.0/nprogress.css](https://unpkg.com/nprogress@0.2.0/nprogress.css) 下載 `nprogress.css` 後將這個檔案放到 `public` 資料夾內，這樣就可以引用這個 `css` 檔

2. 將 `nprogress.min.css` 放到 CDN 上然後在 `Head` 元件內引用（可以參考我們的 `_document.jsx` 檔）：

```HTML
<link
    rel="stylesheet"
    href="https://storage.googleapis.com/builderbook/nprogress.min.css"
/>
```

你的 `href` 會跟我們的有所不同，視你上傳的位址而定。

我們會建議第二個方法，將靜態檔案透過 CDN 發布，通常效能上是比較好的。

`yarn dev` 然後造訪 `http://localhost:8000/login`，點擊左上的 logo 以及右上的`登入`連結，你會看到當路由改變的時候會有個藍色的進度條。

到這裡，我們定義了 `withAuth` HOC 以及實作了 `Login` 頁面以及做了一些使用者體驗上的優化。在我們開始相對來說較為複雜的 Google OAuth API 整合之前，我們稍微離開實作的部分，並且探討一下 `Promise` 以及 `async/await` 語法。如果你對這些概念熟悉，可以跳過下一小節。

---

### 非同步（asynchronous）執行及 callback

以前的 JavaScript 需要透過 `callback` 來確認一個非同步函式（asynchronous function）是否有執行完畢（[https://stackoverflow.com/questions/824234/what-is-a-callback-function](https://stackoverflow.com/questions/824234/what-is-a-callback-function)）。非同步函式簡單來說就是一個需要一定延遲才會完成的函式。而 callback 則是一個函式，而被以參數傳入非同步函式。

附帶一提，callback 並非只有在非同步執行的時候才能使用，也可以用來在同步執行的程式裡面使用，不過我們在這小節只探討非同步函式。

我們在整合 Google OAuth API 時會寫兩個 callback 函式（`verify` 及 `verified`）。

非同步執行的處理主要有三個方式，依據它們在 JavaScript 裡被支援的順序依序為：

1. 非同步 callback（asynchronous callback）
2. `Promise` - 透過 `Promise.then`、`Promise.catch`及 promise 鏈結（promise chaining）
3. `async/await`

從第一個方式來看，當非同步函式完成時 - callback 函式會被執行。因此，你透過 callback 函式的實際執行來判斷非同步函式的結果（透過 callback 來檢查執行是否成功）。

以下是一個 callback 函式的範例：

[https://www.freecodecamp.org/news/javascript-from-callbacks-to-async-await-1cc090ddad99/](https://www.freecodecamp.org/news/javascript-from-callbacks-to-async-await-1cc090ddad99/)

```JavaScript
doThis(andThenThis);

function andThenThis() {
    console.log('and then this');
}

function doThis(callback) {
    console.log('this first');

    callback();
}
```

到你的瀏覽器打開 `Console`，貼上上面的程式碼，按 `Enter`：

![alt Callback](./3-images/3-console-callback.png)

我們將 `andThenThis` 這個 callback 函式作為參數傳給另外一個函式（`doThis`）。可以看到 `andThenThis` 並不會馬上值執行，是直到 `doThis` 執行到最後才被呼叫。

這樣的程式碼很容易就會陷入一個失控不好管理的狀態，下面就是個例子：

[https://www.freecodecamp.org/news/how-to-deal-with-nested-callbacks-and-avoid-callback-hell-1bc8dc4a2012/](https://www.freecodecamp.org/news/how-to-deal-with-nested-callbacks-and-avoid-callback-hell-1bc8dc4a2012/)

```JavaScript
const makeBurger = (nextStep) => {
    getBeef(function(err, beef) {
        if (err) throw err;
        cookBeef(beef, function(err, cookedBeef) {
            if (err) throw err;
            getBuns(function(err, buns) {
                if (err) throw err;
                putBeefBetweenBuns(buns, beef, function(err, burger) {
                    if (err) throw err;
                    nextStep(burger);
                });
            });
        });
    });
};

makeBurger(function (burger) => {
    serve(burger);
});
```

可以看到一樣的錯誤處理不斷出現，這樣的程式碼也往往被戲稱為“災難金字塔（pyramid of doom）”及“callback 地獄（callback hell）”。

---

#### Promise.then

這種地獄開發生活在 Promise 的引進改善了不少。Promise 除了改善了可讀性之外，更透過非同步的 `then/catch` 及 promise 鏈結（promise chaining）讓程式簡潔不少。

上面的 callback 地獄範例改用 `Promise.then` 來寫會變成：

```JavaScript
const makeBurger = () => {
    return getBeef()
        .then((beef) => cookBeef(beef))
        .then(() => getBuns())
        .then((cookedBeef, buns) => putBeefBetweenBuns(cookedBeef, buns))
        .catch(err => throw err);
}

makeBurger().then(burger => serve(burger));
```

Promise 本身就是專為非同步執行所設計的，應該可以感受到它與非同步 callback 相比之下的一些優點：

- 處理錯誤相對簡單很多。只要最後用一個 `Promise.catch` 函式處理所有的錯誤，可以避免多個 `if (err) throw err;`
- 多個非同步 callback 函式可以由 `then` 函式取代，由之前執行的結果透過 `then` 傳入到下個函式，這也就是所謂的 promise 鏈結

我們在透過下面這個範例對 Promise 多加說明：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => resolve("Resolved"), 2000);
});

delay.then(
    result => alert(result),
    error => alert(error)
);
```

到瀏覽器的 `Console`，將上面的程式碼貼上並執行，兩秒鐘過後：

![alt Promise](./3-images/3-console-promise.png)

將程式碼稍微調整一下：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => reject("Rejected"), 2000);
});

delay.then(
    result => alert(result),
    error => alert(error)
);
```

![alt Promise Reject](./3-images/3-console-promise-2.png)

從上面這個例子，可以看到 Promise 與非同步 callback 相似之處：

- `delay` 這個 Promise 需要時間（非同步）完成，並且會傳入一個 callback 函式作為參數

- `delay.then` 會**等待結果**，只有在 `delay` 這個 Promise 有值的時候才會執行（callback 也是要等到主函式執行完畢後才會被呼叫）

- `delay.then` 函式可以取得 `delay` Promise 的 `state` 及 `result` 屬性。`delay.then` 會根據 `state` 及 `result` 來決定執行的是 `alert(result)` 或是 `alert(error)`

在 JavaScript 裡，Promise 是一個特別的物件，這個物件有 `state` 及 `result` 屬性：

- Promise 物件的起始值是 `state: "pending", result: undefined`
- 當 `state: "fulfilled", result: "Resolved"` 時，Promise 會呼叫 `resolve("Resolved")`
- 當 `state: "rejected", result: "Rejected"` 時，Promise 會呼叫 `reject("Rejected")`

請注意，Promise 物件的 `state` 及 `result` 是**無法透過**程式碼取得，只有 `Promise.then` 這個函式能夠存取。

至於 promise 鏈結的好處在當需要讓多個函式依序地在前一個函式執行完畢時才執行接下來的函式可以顯現出來，可以到 `Console` 執行下列的程式：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => resolve("Resolved"), 2000);
});

delay
    .then(result => {
        alert(result);
        return result;
    })
    .then(
        result => alert(`${result} again`),
    );
```

執行上面的例子會依序看到兩個 `alert`。第一個會在執行程式後兩秒發生，顯示 `Resolved`。當你一點選 `OK` 之後會馬上出現 `Resolved again` 的第二個 `alert`。第二個 `alert` 只有在第一個 Promise 真的有值的時候（也就是你點選 `OK`）才會執行，這就是使用 Promise 鏈結可以協助確保程式碼執行順序的優點。

`Promise.then` 應該比較清楚了，但是什麼是 `Promise.catch` 呢？試著執行以下的程式碼：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => reject("Rejected"), 2000);
});

delay.then(
    result => alert(result)
);
```

你將不會看到任何 `alert`，反而是在終端機會看到 `Uncaught` 的 `error`：

![alt Promise Reject](./3-images/3-console-uncaught-reject.png)

在 `delay.then` 的第一個參數加上 `null`，再跑一次：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => reject("Rejected"), 2000);
});

delay.then(
    null, // 負責處理 success 的函式
    result => alert(result) // 負責處理 error 的函式
);
```

這次你在兩秒後會看到 `Rejected` 的 `alert` - 你正確地捕捉到 `error` 了。而 `Promise.catch(function)` 就是 `Promise.then(null, function)` 的簡潔寫法。因此上面的程式碼可以改成：

```JavaScript
let delay = new Promise((resolve, reject) => {
    setTimeout(() => reject("Rejected"), 2000);
});

delay.catch(
    result => alert(result) // catch 就專職負責處理 error 的函式
);
```

Promise 所具備的類別函式以及實體函式（static and instance methods）可以參考 JavaScript 的官方文件：

[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

Promise 本身絕對是個進步，而繼 Promise 之後 JavaScript 推出的 `async/await` 這個語法糖（syntactic sugar）又更加讓程式更簡潔、可讀。

---

#### async/await

我們來將前面 `makeBurger` 的 Promise 範例用 `async/await` 重寫：

```JavaScript
const makeBurger = () => {
    return getBeef()
        .then((beef) => cookBeef(beef))
        .then(() => getBuns())
        .then((cookedBeef, buns) => putBeefBetweenBuns(cookedBeef, buns))
        .catch(err => throw err);
}

makeBurger().then(burger => serve(burger));
```

會變成：

```JavaScript
async makeBurger() {
    try {
        const beef = await getBeef();
        const cookedBeef = await cookBeef(beef);
        const buns = await getBuns();
        const burger = await putBeefBetweenBuns(cookedBeef, buns);
    } catch (err) {
        throw err;
    }

    return burger;
}

serve(await makeBurger());
```

這樣的改動，讓程式碼跟同步執行的程式幾乎維持一致的寫法，可讀性大幅提升。

Node.js 自 version 7.6 開始就支援了 `async/await`，目前已經算是相當受歡迎與普及的用法。我們本專案原則上都會使用 `async/await` 而非 Promise 來處理非同步執行的程序。原則上所有的 `Promise.then` 都可以使用 `async mainFunction` 與 `await Promise` 這樣的對應方式來應用（註：還是有些例外，例如陣列的 `forEach` 函式，就不好使用 `async/await`，而主要都是使用 `Promise.all` 這樣的方式處理）。

我們本章晚點會在我們的 `User` 資料模型上新定義靜態函式（static methods）。而在第五章我們會在 `Book` 資料模型上定義它的靜態函式。為了讓我們對 `async/await` 可以再多理解，讓我們看一下 `Book` 資料模型的 `Book.add` 函式（簡化版，完整版在第五章）。

`Book.add` 這個靜態函式會使用 `server/utils/slugify.js` 的 `generateSlug` 函式來產出一個 `slug`（舉例來說，書名 `Good Book` 會被 `generateSlug` 產出一個對應的 `slug`： `good-book`）。`Book.add` 接著會使用 `create` 這個 Mongoose API 來建立一個新的 `Book` MongoDB 文件。如果我們使用 Promise 來實作 `Book.add`，它大概會像以下：

```JavaScript
static add({ name, price, githubRepo }) {
    return generateSlug(this, name).then(slug =>
        this.create({
            slug,
            // 其他需要新增書本的參數省略
        })
    );
}
```

Mongoose API `create` 只有在 `generateSlug` 所回傳的 Promise 具有值（也就是 `slug`）的時候才會進行 `then` 裡的內容。

使用 `async/await` 的話，以下會有一樣的效果：

```JavaScript
static async add({ name, price, githubRepo }) {
    const slug = await generateSlug(this, name);
    return this.create({
        slug,
        // 其他需要新增書本的參數省略
    })
}
```

我們在 `add` 前加上 `async` 然後在 `generateSlug` 加上 `await` 就可以確保會拿到 Promise 所回傳的值。`await` 會告知 JavaScript 說要等待 `generateSlug` 這個函式非同步執行完畢，才將結果存放到 `slug` 裡。

相信多數人都會同意這是比較可讀的寫法。

---

我們來做一些實驗來說明 `async` 的行為。在瀏覽器的 `Console` 裡執行以下的程式：

```JavaScript
let foo = () => {
    return 'Hello world!';
}

foo().then(alert);
```

![alt Promise Then](./3-images/3-console-then.png)

這是正確的，因為 `foo()` 回傳的不是 Promise（而是一個字串，字串當然沒有 `then` 這個函式），因此不能拿 `foo().then` 來嘗試處理 `result/error`。

將 `async` 給加到上面的函式：

```JavaScript
let foo = async () => {
    return 'Hello world!';
}

foo().then(alert);
```

![alt Async](./3-images/3-console-async.png)

這次就會**顯示 alert**了。這裡主要是說明，當將函式定義加上 `async` 的時候，JavaScript 會主動將函式的回傳值用 Promise 給包覆起來，確保回傳值一定是一個 Promise。即使 `foo` 看起來像是回傳字串 `return 'Hello world!'`，實際上 JavaScript 會把這個回傳值以 Promise 包覆起來，並以 `resolved` 狀態回傳。

---

接著我們看看 `await` 的部分。`await` 的特色則是在 `async` 函式內（不可在 `async` 函式外或是在非 `async` 函式內 - 違反規定的時候 ESLint 會警告你），你可以在回傳 Promise 的函式前面加上 `await` 以達到與同步執行極為類似的行為。

用我們上面的 `add` 靜態函式來看，JavaScript 會在 `await` 那行*暫停執行*（也就是下面這行）：

```JavaScript
const slug = await generateSlug(this, name);
```

JavaScript 會等待對應的 Promise 完成（也就是 `Promise.state` 變成 `"fulfilled"` 或是 `"rejected"`）並回傳 `result`（在本範例就是書名的 `slug` 值）。

在實務上，`async/await` 在許多地方都可以幫上忙。舉例來說，所有的 Mongoose API 都是回傳 Promise（`create`、`update`、`findOne` 等）。我們通過 `await` 這些 Mongoose API 就可省略很多 `new Promise((resolve, reject) => ...)` 這樣的程式碼了。

---

值得一提的是 `await` 所對應的函式必須要是一個回傳 Promise 的函式才會有*暫停執行*的效果，讓我們用下面兩個範例來說明：

```JavaScript
let delay = async () => {
    let foo = new Promise((resolve, reject) => {
        setTimeout(() => resolve('Resolved'), 2000)
    });

    console.log('Line BEFORE await');
    let result = await foo;
    console.log('Line AFTER await');

    alert(result);
}

delay();
```

![alt Async](./3-images/3-promise-await.png)

注意一下整個過程的執行順序：

1. 終端機印出 `Line BEFORE await`
2. 有約兩秒鐘沒有執行動作 - 這是 JavaScript 等待 `await foo` 的 Promise 被 resolve
3. `Line AFTER await` 與 `alert` 同時出現（嚴格來說，`console.log` 可能比較快個幾毫秒）

如果我們的 `foo` 並非回傳 Promise：

```JavaScript
let delay = async () => {
    let foo = setTimeout(() => alert("Resolved"), 2000);

    console.log('Line BEFORE await');
    let result = await foo;
    console.log('Line AFTER await');

    alert(result);
}

delay();
```

這次的執行順序：

1. `Line BEFORE await` 與 `Line AFTER await` 沒延遲的依序印出
2. 兩秒後才有 `alert`

以上的範例正是在表達說，如果 `await` 對應的函式並非回傳 Promise 時，則不會等待，會直接依序執行接下來的程式碼。

另外個重點是處理錯誤的方式，Promise 是透過 `Promise.then` 之後的 `Promise.catch`，而 `async/await` 是用 `try/catch` 來處理錯誤：

[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch)

以下是節錄我們未來會寫的 `/books` Express 路由作為範例：

```JavaScript
router.get('/books', async (req, res) => {
    try {
        const books = await Book.list(); // Book.list() 假設 throw error 會被 catch 住
        res.json(books);
    } catch (err) { // 這裡會 catch 住 await 對應函式所拋出的 error
        res.json({ error: err.message || err.toString() });
    }
});
```

總結一下，`async/await` 可以達成幾乎所有 Promise 及其函式所具備的功能。但是其可讀性是好很多的。我們原則上會盡量使用 `async/await` 來進行非同步的執行作業。

---

### 整合 Google OAuth API

到了這裡，我們介紹了 MongoDB 資料庫，並討論了 `session` 及 `cookie` 的概念了。但是我們還沒有真的實作登入的功能。現在總算要進入主題了。我們在本書會使用 Google OAuth API 來進行使用者的認證，這代表的意思是，使用者可以使用他的 Google 帳號登入進我們的網頁應用程式。

使用第三方的 API 是個雙面刃。你會失去架構上的自主性與掌控權，但是登入流程對很多使用者來說會是熟悉而且簡單的。身為開發者，你必須要祈禱 Google 不要改版 OAuth API 的時候導致錯誤，更要祈禱他們的 API 服務不要哪一天被關閉了。你對這些是沒有任何掌控權的。但反觀來說，許多使用者是信任 Google，且安全性往往相對來說相當高，例如有雙因素認證（two factor authentication）。另外，使用使用者已經習慣的登入方式對於客戶黏著度是很有效力的。

在本節中，我們會使用 `passport` 及 `passport-google-oauth` 這兩個由 Jared Hanson 開發的套件：

[https://github.com/jaredhanson](https://github.com/jaredhanson)

以下是 Google OAuth 的流程，我們在本章也就是會依據這個流程跟架構來實作：

1. 使用者造訪我們的 `Login` 頁面，並點擊 `使用 Google 登入`
2. 瀏覽器會發出一個造訪 `/auth/google` 的請求，我們的伺服器對應的 Express 路由會處理
3. 我們伺服器的 `/auth/google` Express 路由內會執行 `passport.authenticate` 並回應請求
4. 我們伺服器回應給瀏覽器的內容中會指示瀏覽器要轉導（到 Google 架設的網頁）
5. 使用者瀏覽器聽話的轉導到一個由 Google 架設的網頁
   - 使用者可以選擇要使用的 Google 帳號
   - 跟 Google 確認登入的意圖
6. 上述動作做完後，瀏覽器會將上述資訊透過 HTTP 請求傳到 Google 的 OAuth 伺服器
7. Google 的 OAuth 伺服器回傳的內容也一樣會指示瀏覽器要轉導（到我們伺服器）
8. 瀏覽器轉導至我們伺服器上的 `/oauth2callback` Express 路由
9. 我們的伺服器會與 Google 的 OAuth 伺服器以 HTTP 溝通，我們的伺服器會取得：
   - Google token
   - 使用者資料（例如，頭像網址、使用者名稱及 email 等）
10.

- 假設認證成功：

  1. 我們的伺服器會呼叫 `verify` 函式，裡面會呼叫 `User.signInOrSignUp` 函式，它會：
     - 回傳對應 MongoDB 存在的對應 `User` 文件
     - 沒對應的 `User` 時，會新增一個 `User`
  2. 我們的伺服器會將使用者資料的物件存放到 `req.user` ，並存放一個唯一的 `Session` 文件到 MongoDB 資料，最後並將對應的唯一 `cookie` 回傳讓使用者的瀏覽器存放
  3. 瀏覽器接者會轉導至 `Index` 頁面（但是現在瀏覽器有正確的 `cookie`，因此就登入成功了）

- 假設認證失敗：

  1. 我們的伺服器就簡單的回傳給瀏覽器轉導至 `Login` 頁面的指示

TODO: Flow Chart

---

我們來一步步看一下，假設現在造訪 `http://localhost:8000/login`，點選 `使用 Google 登入` 這個按鈕，你會看到一個 `404 頁面`：

![alt 404](./3-images/3-login-404.png)

為什麼會這樣呢，可以看一下 `/pages/login.jsx` 這個按鈕的定義：

```JSX
<Button variant="contained" style={styleLoginButton} href="/auth/google">
    <img
    src="https://storage.googleapis.com/builderbook/G.svg"
    alt="使用 Google 登入"
    style={{ marginRight: '10px' }}
    />
    使用 Google 登入
</Button>
```

這個按鈕嘗試著對 `/auth/google` 發出 HTTP 請求，而我們的伺服器並沒有處理這個路由的邏輯，因此 Next.js 會用一個預設的 404 頁面顯示。

而處理 `/auth/google` 的邏輯我們將會實作於 `/server/google.js` 裡面，這個檔案會 `export` 出一個 `setupGoogle` 函式，由 `/server/server.js` 引用並呼叫來處理 `/auth/google` 這個路由所需執行的事項。

我們先來把相關檔案加到專案內然後逐步地完成各事項：

- 新增一個 `/server/google.js` 檔案，內容先很單純：

```JavaScript
function setupGoogle({ server, ROOT_URL }) {
  // 接下來需要將 Google 認證相關步驟給實作出來
}

module.exports = setupGoogle;
```

- 在 `/server/server.js` 裡面引用跟呼叫 `setupGoogle`：

```JavaScript
// 省略
const setupGoogle = require('./google');
const User = require('./models/User');

// 省略
app.prepare().then(() => {
    // 省略
    server.use(session(sess));
    setupGoogle({ server, ROOT_URL });
    // 省略
}
```

當然現在點擊 `使用 Google 登入` 仍然是會看到 404 頁面，因為我們什麼都還沒做，讓我們接著走下去。

---

#### setupGoogle 函式、verify 函式、passport 及 strategy

在這小節中，我們會針對 `server/google.js` 裡面定義的 `setupGoogle` 函式進行實作。我們是在伺服器（`server/server.js`）引用並且呼叫這個函式。

`setupGoogle` 函式的作用為何？我們透過這個函式要達成下面幾項事情：

- 定義護照及護照策略（passport 及 passport strategy）
- 定義會呼叫 `User.signInOrSingUp` 的 `verify` 函式
- 將使用者資訊序列化／去序列化（serialize/deserialize）- 換言之，序列化指的是將使用者 id 存入 `session`；去序列化是將 `session` 內的使用者 id 取出，以用來查詢對應的 `User` MongoDB 文件。要將 `user` 物件存入 `req.user` 這兩個動作都是必要的
- 定義三個 Express 路由： `/auth/google`、`/auth/google/callback` 及 `/logout`
- 在 `server/server.js` 內引用並呼叫 `setupGoogle` 函式

我們會參考 `passport` 及 `passport-google-ouath` 的官方範例來定義 `setupGoogle`:

[https://github.com/jaredhanson/passport-google-oauth2#configure-strategy](https://github.com/jaredhanson/passport-google-oauth2#configure-strategy)

套件作者的範例中，用以下的策略（strategy）設定來當作傳入 `passport.use` 函式的參數，並用來建立 `passport` 實體:

```JavaScript
passport.use(new GoogleStrategy({
    clientID: GOOGLE_CLIENT_ID,
    clientSecret: GOOGLE_CLIENT_SECRET,
    callbackURL: "http://www.example.com/auth/google/callback"
  },
  function(accessToken, refreshToken, profile, cb) {
    User.findOrCreate({ googleID: profile.id }, function(err, user) {
      return cb(err, user);
    });
  }
));
```

`function(accessToken, refreshToken, profile, cb)` 這個匿名函式（anonymous function）正就是我們的 `verify` callback 函式。這個函式會從 Google 的 OAuth 伺服器**取得** token 以及使用者資訊。為了讓程式碼更好讀，我們將之取名為 `verify`。

從第二章開始，我們就是在使用 `dotenv` 套件來管理環境變數。我們透過 `process.env.xxx` 這樣的方式來取得 `xxx` 這個環境變數的值。舉例來說，我們之前的程式碼裡的 `server/server.js` 的 `MONGO_URL` 就是用 `const MONGO_URL = process.env.MONGO_URL_TEST` 來設定。所以我們可以將上面的 `GoogleStrategy` 相關的設定用如下的方式定義:

TODO: confirm if it is consistent with .env

```JavaScript
{
  clientID: process.env.GOOGLE_CLIENTID,
  clientSecret: process.env.GOOGLE_CLIENTSECRET,
  callbackURL: `${ROOT_URL}/oauth2callback`
}
```

以這些環境變數以及 `verify` 來建立 `Strategy` 這個物件，指定這個 `Strategy` 作為 `passport.use` 所使用的認證策略。我們 `server/google.js` 的 `setupGoogle` 函式現在變成如下:

```JavaScript
const passport = require('passport');
const Strategy = require('passport-google-oauth').OAuth2Strategy;

function setupGoogle({ server, ROOT_URL }) {
  // 根據 passport 範例，我們先需要引入並且 use 這個套件
  passport.use(
    new Strategy(
      {
        clientID: process.env.GOOGLE_CLIENTID,
        clientSecret: process.env.GOOGLE_CLIENTSECRET,
        callbackURL: `${ROOT_URL}/oauth2callback`,
      },
      verify, // 接著需要定義跟實作這個函式
    ),
  );
}

module.exports = setupGoogle;
```

我們來看一下 `setupGoogle` 函式內的幾個重點:

- `verify` 是個 callback 函式，`passport` 在從 Google 伺服器取得由 Google 提供的 `accessToken`、`refreshToken` 以及 `profile` 資料後而被 `passport` 呼叫。`passport` 套件規範這個 `verify` 函式必須自己內含另外一個 callback 函式，我們將這個定名為 `verified`（這可以是任何名字，值得注意的是這個函式是由 `passport` 所提供的）。因此 `verify` 的函式原型（function signature）會是:

```JavaScript
const verify = async (accessToken, refreshToken, profile, verified) => { ... }
```

上面的 `profile` 物件裡面可以取得以下的 Google 帳號資訊:

- `googleId` (`profile.id`)
- `email` (`profile.emails[0].value`)
- `displayName` (`profile.displayName`)
- `avatarUrl` (`profile.image.url`)

可以看到 `profile.emails` 是一個陣列。也可以看到我們簡單的選取陣列的第一筆 email 來使用:

```JavaScript
if (profile.emails) {
  email = profile.emails[0].value;
}
```

我們另外會將頭像圖片的大小透過改變 `profile.image.url` 結尾從 Google 預設的 `?sz=50` 改為 `?sz=128`:

```JavaScript
if (profile.photos && profile.photos.length > 0) {
  avatarUrl = profile.photos[0].value.replace('sz=50', 'sz=128');
}
```

可以在 Mozilla 文件參閱 JavaScript 的 `replace` 函式相關的資訊:

[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace)

將上面兩段工作放進 `verify` 函式定義中，會得到:

```JavaScript
// passport 所需要使用的 callback 函式
const verify = async (accessToken, refreshToken, profile, verified) => {
    let email;
    let avatarUrl;

    if (profile.emails) {
        email = profile.emails[0].value;
    }

    if (profile.photos && profile.photos.length > 0) {
        avatarUrl = profile.photos[0].value.replace('sz=50', 'sz=128');
    }

    // 利用從 Google 拿到的 profile，確認 MongoDB 是否有此 user，假設沒有，註冊該 user
};
```

- 拿到 Google 的 `profile` 後呼叫 Mongoose API 的 `User.signInOrSignUp` 來取得回傳的 `user`，而因為 `signInOrSignUp` 回傳的是一個 Promise，所以我們使用 `await`:

```JavaScript
const user = await User.signInOrSignUp();
```

我們需要將 Google OAuth 伺服器回傳的資訊當參數傳入 `User.signInOrSignUp`:

```JavaScript
const user = await User.signInOrSignUp({
    googleId: profile.id,
    email,
    googleToken: { accessToken, refreshToken },
    displayName: profile.displayName,
    avatarUrl,
});
```

我們也應該要捕捉 `error` 以防有錯誤發生。如同前面所介紹，我們將 `try/catch` 與 `async/await` 結合使用:

TODO: check code

```JavaScript
try {
    const user = await User.signInOrSignUp({
    googleId: profile.id,
    email,
    googleToken: { accessToken, refreshToken },
    displayName: profile.displayName,
    avatarUrl,
    });
    verified(null, user);
} catch (err) {
    verified(err);
    console.log(err);
}
```

根據官方文件:

[http://www.passportjs.org/docs/google/](http://www.passportjs.org/docs/google/)

`verified`（官方文件是稱之為 `done`）這個 callback，函式原型為:

```JavaScript
verified(error, user, info)
```

值得提的是，這個 `verified` 是由 `passport` 提供傳入我們的 `verify`，這塊新手往往會有點混淆。

- 當 `verify` 是成功的時候，我們 `error` 參數會對應傳入 `null`，如: `verified(null, user)`

- 當 `verify` 是失敗的時候，我們將 `null` 傳給 `user` 參數，如: `verified(err, null)` 或簡化後可以是 `verified(err)`

結合完畢，你的 `verify` 函式應該如下:

```JavaScript
const verify = async (accessToken, refreshToken, profile, verified) => {
    let email;
    let avatarUrl;

    if (profile.emails) {
      email = profile.emails[0].value;
    }

    if (profile.photos && profile.photos.length > 0) {
      avatarUrl = profile.photos[0].value.replace('sz=50', 'sz=128');
    }

    try {
      const user = await User.signInOrSignUp({
        googleId: profile.id,
        email,
        googleToken: { accessToken, refreshToken },
        displayName: profile.displayName,
        avatarUrl,
      });
      verified(null, user);
    } catch (err) {
      verified(err);
      console.log(err);
    }
};
```

- 想像一下，假設有個使用者，登入了我們的網頁，關閉頁籤，重新開啟頁籤，他將看到他仍然是處於登入的狀態。這正是我們第二章提到的永久性登入的機制。快速重點複習，達到這個功能的方式是透過比對確認瀏覽器端的 `cookie` （解譯後）有對應的 `session` （存於我們 MongoDB 的 `Session` 文件）。附帶一提，`session` 物件內會包含使用者唯一的 user id。

`passport` 透過將 `user.id` 存到 session 內來將兩者的關聯建立起來。它使用的是 `passport.serializeUser` 這個函式:

```JavaScript
passport.serializeUser((user, done) => {
    done(null, user.id)
})
```

可以注意到 `done` 的用法跟 `verified` 一樣，將 `null` 傳到對應 `error` 的參數。

我們目前 MongoDB 上的 `bookstore.sessions` 集合內的 `Session` 文件如下：

```JSON
{
    "_id": "2jaiZFzClBbjMMq5s1DiBoM19h6_v-aF",
    "expires": {
        "$date": {
            "$numberLong": "1646627821387"
        }
    },
    "session": "{\"cookie\":{\"originalMaxAge\":1209600000,\"expires\":\"2022-02-25T03:52:21.572Z\",\"httpOnly\":true,\"domain\":\"localhost\",\"path\":\"/\"},\"foo\":\"bar\"}"
}
```

而在透過 `passport.serializeUser` 後加入 MongoDB 的 `Session` 文件會變成：

```JSON
{
    "_id": "2jaiZFzClBbjMMq5s1DiBoM19h6_v-aF",
    "expires": {
        "$date": {
            "$numberLong": "1646627821387"
        }
    },
    "session": "{\"cookie\":{\"originalMaxAge\":1209600000,\"expires\":\"2022-02-25T03:52:21.572Z\",\"httpOnly\":true,\"domain\":\"localhost\",\"path\":\"/\"},\"foo\":\"bar\",\"passport\":{\"user\":\"5a6e33134989d700c5fba909\"}}"
}
```

從範例可以看到，`passport.serializeUser` 會將資料以 `session.passport.user` 屬性的方式將資料存入 `Session` MongoDB 文件。我們將 `session` 這個屬性的字串去序列化後會如下，比較清楚：

```JSON
{
    "session": {
        "cookie": {
            "originalMaxAge": 1209600000,
            "expires": "2022-02-25T03:52:21.572Z",
            "httpOnly": true,
            "domain": "localhost",
            "path": "/"
        },
        "foo": "bar",
        "passport": {
            "user": "5a6e33134989d700c5fba909"
        }
    }
}
```

- 當成功登入（也沒有進行登出）的使用者載入我們的頁面時，會發生甚麼事? 我們的伺服器需要將使用者資訊給**去序列化**並取出來。如果從瀏覽器來的 `cookie` 與 `session` 對應正確，且 `session` 存有使用者 id，那我們需要這個 id 取出並透過 `User.findById` 這個 Mongoose API 去確認這個使用者 id 有存在於 `User` MongoDB 集合裡。基於安全性的考量，我們只將 `public` 的欄位回傳給瀏覽器。而我們可以透過呼叫 `User.publicFields` 函式來取得 `public` 的欄位。

而找到該 id 的使用者之後，`passport` 透過 `passport.deserializeUser` 函式來將資料存入 `req.user`:

```JavaScript
passport.deserializeUser((id, done) => {
    User.findById(id, User.publicFields(), (err, user) => {
      done(err, user);
    });
});
```

說到這裡必須要探討一下 `req.user` 的重要性。

`passport` 會在每次登入或註冊時，在我們的伺服器端建立 `req.user` 的資料。從上面的程式碼片段可以看到，`req.user` 內含著一個具有 `public` 屬性的 `user` 物件。我們專案在兩個重要的地方會使用 `req.user`:

- `withAuth` HOC 會在 `getInitialProps` 函式使用 `req.user`。`withAuth` HOC 接著會將 `user` 以 prop 的方式傳遞到它所包覆的頁面

- 我們在數個 Express 路由以及 Express 中介層（middleware）內會使用 `req.user`

  TODO: add more description on req.user

TODO: add more description on middleware 4. 第二章我們有提到 `server.use` 是一個仲介層函式。我們之前有在 `server/server.js` 裡面使用，如 `server.use(session(sess));`。

而 `passport` 需要透過中介層來掛載 `passport.initialize` 才能針對 Express 路由進行認證的動作:

```JavaScript
server.use(passport.initialize());
```

而 `passport` 則是透過掛載 `passport.session` 來達到永久性的登入 session:

```JavaScript
server.use(passport.session());
```

- 我們在 `server/server.js` 裡引入並且呼叫 `setupGoogle`，而這會將所有的 Express 中介層以及 Express 路由掛載到我們的 Express 伺服器上。我們會在第五章更加詳細地探討 Express 中介層、路由器以及路由

總結一下登入的使用者在造訪 `Index` 頁面時會發生的事情：

1. 使用者打開頁籤
2. 使用者的瀏覽器透過 HTTP 傳遞 request 到伺服器
   - 這個 request 含有 `headers`，而其中具有 `cookie`
3. 伺服器端將 `req.headers.cookie` 解譯出唯一的 `cookie` 並且尋找對應的 `session`（第二章有解說）
4. 所找到的對應 `session` 內含有 user id（這個 id 是透過初次登入時所執行的 `serializeUser` 函式所存放的）
5. Passport 透過 id 找到 `user`，將資訊經由 `deserializeUser` 函式存入到 `req.user`
6. `withAuth` 使用 `req.user` 來將 `user` prop 傳入到 `Index` 頁面

我們 `setupGoogle()` 快完成了，只差將 Express 路由也進行設定，目前的 `/server/google.js` 檔案如下：

```JavaScript
const passport = require('passport');
const Strategy = require('passport-google-oauth').OAuth2Strategy;

const User = require('./models/User');

function setupGoogle({ server, ROOT_URL }) {
  // passport 所需要使用的 callback 函式
  const verify = async (accessToken, refreshToken, profile, verified) => {
    let email;
    let avatarUrl;

    if (profile.emails) {
      email = profile.emails[0].value;
    }

    if (profile.photos && profile.photos.length > 0) {
      avatarUrl = profile.photos[0].value.replace('sz=50', 'sz=128');
    }

    try {
      const user = await User.signInOrSignUp({
        googleId: profile.id,
        email,
        googleToken: { accessToken, refreshToken },
        displayName: profile.displayName,
        avatarUrl,
      });
      verified(null, user);
    } catch (err) {
      verified(err);
      console.log(err);
    }
  };

  // 根據 passport 範例，我們先需要引入並且 use 這個套件
  passport.use(
    new Strategy(
      {
        clientID: process.env.GOOGLE_CLIENTID,
        clientSecret: process.env.GOOGLE_CLIENTSECRET,
        callbackURL: `${ROOT_URL}/oauth2callback`,
      },
      verify,
    ),
  );

  // 將 user.id 存入 session 內
  passport.serializeUser((user, done) => {
    done(null, user.id);
  });

  // 將 user 去序列化，檢查 MongoDB 是否存有此 user
  passport.deserializeUser((id, done) => {
    User.findById(id, User.publicFields(), (err, user) => {
      done(err, user);
    });
  });

  server.use(passport.initialize());
  server.use(passport.session());

  // 設定 Express 路由
}

module.exports = setupGoogle;

```

我們在接下來的小節來看一下所新增的三個 Express 路由。

---

#### /auth/google、/oauth2callback 及 /logout Express 路由

我們在這節中會定義 `setupGoogle` 函式該具備的 Express 路由，這樣就完成了我們的 `server/google.js` 檔案了。我們第五章會再針對 Express 進行更詳盡的討論，這章我們只是建立基本的了解，並且透過實際的程式碼先抓一些感覺。

目前你只需要知道幾件事：

- Express 路由只會在伺服器端執行
- 用來接受發往某 API 端點（API endpoint，在 Express 體系內慣用語就是路由）的 HTTP request
- 路由內會呼叫處理函式（handler function）來回傳 HTTP response
- 最常見的 HTTP request/response 互動是瀏覽器與伺服器間的溝通，但是也有可能是伺服器與伺服器間 API 的溝通

我們來看個簡單的 Express 路由範例：

```JavaScript
app.get('/', function (req, res) {
    res.send('hello world')
})
```

`.get` 的部分表示這個路由是處理 `req.method = "GET"` 的 request。`app.get()` 裡面的兩個參數，第一個指定的是 `path` 的值，也就是路由的位址，第二個參數則是上述的處理函式。在多數情況下，這個處理函式會從 `req` 取出需要的資訊，進行處理後，再透過 `res` 回傳結果。

我們在 `setupGoogle()` 定義三個 Express 路由，這會讓伺服器知道如果有造訪這些路由的 request 時，該做什麼動作：

- `/auth/google`
- `/oauth2callback`
- `/logout`

可以看一下 `passport` 套件進行 Google 認證的官方範例：

[https://github.com/jaredhanson/passport-google-oauth2#authenticate-requests](https://github.com/jaredhanson/passport-google-oauth2#authenticate-requests)

我們實際的程式跟官方的範例會很接近，我們先來把框架加到 `/server/google.js` 裡：

```JavaScript
server.use(passport.initialize());
server.use(passport.session());

// 這是我們 Login button 會造訪的路由
server.get(
    '/auth/google',
    passport.authenticate('google', {
      // 將 options 傳到 passport.authenticate 內
    }),
);

// 這是 Google OAuth server 會造訪的路由
server.get(
    '/oauth2callback',
    passport.authenticate(
      'google',
      {
        failureRedirect: '/login',
      },
      (req, res) => {
        // 執行到這邊的話代表認證成功，因此將使用者轉導回 Index 頁面('/')
      },
    ),
);

// 這是使用者手動登出會造訪的路由
server.get('/logout', (req, res) => {
    // 將 req.user 以及 session 內的 user id 清空，轉導回 Login 頁面('/login')
});
```

注意一下，我們是根據 Google Cloud 文件建議而使用 `/oauth2callback` 這個路由名稱。不過實際上你可以使用任何名稱，只要待會於 Google Cloud 上設定時候指定正確即可。

- `/auth/google` 路由上，我們希望存取 Google 使用者資料的兩個範圍（scope）：`userinfo.profile`及`userinfo.email`。下面有 `scope` 裡 `profile` 以及 `email` 的 option 資訊：

[https://developers.google.com/+/web/api/rest/oauth#profile](https://developers.google.com/+/web/api/rest/oauth#profile)

[https://developers.google.com/+/web/api/rest/oauth#email](https://developers.google.com/+/web/api/rest/oauth#email)

如果對 `scope` 及 `prompt` 的所有選項有興趣：

[https://developers.google.com/identity/protocols/OpenIDConnect#scope-param](https://developers.google.com/identity/protocols/OpenIDConnect#scope-param)

因此我們的 `/auth/google` Express 路由會如下：

```JavaScript
server.get(
    '/auth/google',
    passport.authenticate('google', {
        scope: ['profile', 'email'],
        prompt: 'select_account',
    }),
);
```

看到這裡，你可能會有個疑問，為什麼這個 Express 路由沒有回傳一個 response？實際上是有的，只是被 `passport.authenticate` 給處理了，變成我們自己不需要去處理跟撰寫。當我們在上面使用 `passport.authenticate` 的時候，`passport` 會組出一個 URL 回傳 response 給瀏覽器要瀏覽器轉導致 `passport` 所組出的 URL 位址。而這個 URL 則是導向一個由 Google 架設的登入頁面。在這個頁面上，使用者可以選擇他想要使用的 Google 帳號，並且進行確認登入的動作。

- 在 Google 的 OAuth 伺服器認證了使用者後，Google 的伺服器會回傳一個 response 給使用者的瀏覽器。這個 response 內會要求轉導至 `/oauth2callback` 這個路由。當使用者的瀏覽器造訪 `/oauth2callback` 時，就會由我們的伺服器處理了。我們的 `/oauth2callback` 路由是參照官方範例，也一樣再次呼叫 `passport.authenticate` 函式，但是這次的參數有所不同。第一個參數仍舊是 `'google'`，第二個參數是一個含有 `failureRedirect` 屬性的物件，而第三個參數則是一個處理函式，我們這次透過明確地（explicitly）的透過處理函式回傳 response 給瀏覽器：

```JavaScript
server.get(
    '/oauth2callback',
    passport.authenticate(
        'google',
        {
        failureRedirect: '/login',
        },
        (req, res) => {
        res.redirect('/');
        },
    ),
);
```

當 Google 的伺服器成功的認證使用者的時候，會觸發我們的處理函式，我們就會將使用者轉導回 `/`，也就是 `Index` 頁面。

當 Google 的伺服器判斷認證失敗的時候，我們會透過第二個參數將使用者導至 `/login`，也就是 `Login` 頁面。

- 我們第三個路由使用的是 `passport` 所提供的一個功能，`passport` 讓我們可以對 `req` 物件呼叫 `req.logout`，可以參考以下：

[http://www.passportjs.org/docs/logout](http://www.passportjs.org/docs/logout)

呼叫 `req.logout` 就會將 `req.user` 屬性設為 null 並且將對應 `session` 物件內的 `user` 屬性移除。

因此 `/logout` Express 路由就變成：

```JavaScript
server.get('/logout', (req, res) => {
    // 將 req.user 以及 session 內的 user id 清空，轉導回 Login 頁面('/login')
    req.logout();
    res.redirect('/login');
});
```

真的有興趣的話，你可以看一下 `passport` 內 `req.logout` 的實作：

[https://github.com/jaredhanson/passport/blob/a892b9dc54dce34b7170ad5d73d8ccfba87f4fcf/lib/passport/http/request.js#L58](https://github.com/jaredhanson/passport/blob/a892b9dc54dce34b7170ad5d73d8ccfba87f4fcf/lib/passport/http/request.js#L58)

```JavaScript
req.logOut = function() {
  if (!this._passport) throw new Error('passport.initialize() middleware not in use');

  var property = this._passport.instance._userProperty || 'user';

  this[property] = null; // 將 req.user 設為 null
  delete this._passport.session.user; // 將 user 從 session.passport.user 移除
};
```

`req.logout` 之後我們透過 Express 提供的 `res.direct` 提供轉導指令的 response：

[https://expressjs.com/en/4x/api.html#res.redirect](https://expressjs.com/en/4x/api.html#res.redirect)

我們的 `server/google.js` 總算完成了！

```JavaScript
const passport = require('passport');
const Strategy = require('passport-google-oauth').OAuth2Strategy;

const User = require('./models/User');

function setupGoogle({ server, ROOT_URL }) {
  // passport 所需要使用的 callback 函式
  const verify = async (accessToken, refreshToken, profile, verified) => {
    let email;
    let avatarUrl;

    if (profile.emails) {
      email = profile.emails[0].value;
    }

    if (profile.photos && profile.photos.length > 0) {
      avatarUrl = profile.photos[0].value.replace('sz=50', 'sz=128');
    }

    try {
      const user = await User.signInOrSignUp({
        googleId: profile.id,
        email,
        googleToken: { accessToken, refreshToken },
        displayName: profile.displayName,
        avatarUrl,
      });
      verified(null, user);
    } catch (err) {
      verified(err);
      console.log(err);
    }
  };

  // 根據 passport 範例，我們先需要引入並且 use 這個套件
  passport.use(
    new Strategy(
      {
        clientID: process.env.GOOGLE_CLIENTID,
        clientSecret: process.env.GOOGLE_CLIENTSECRET,
        callbackURL: `${ROOT_URL}/oauth2callback`,
      },
      verify,
    ),
  );

  // 將 user.id 存入 session 內
  passport.serializeUser((user, done) => {
    done(null, user.id);
  });

  // 將 user 去序列化，檢查 MongoDB 是否存有此 user
  passport.deserializeUser((id, done) => {
    User.findById(id, User.publicFields(), (err, user) => {
      done(err, user);
    });
  });

  // 我們第五章會多說明 use 等用法
  server.use(passport.initialize());
  server.use(passport.session());

  server.get(
    '/auth/google',
    passport.authenticate('google', {
      scope: ['profile', 'email'],
      prompt: 'select_account',
    }),
  );

  server.get(
    '/oauth2callback',
    passport.authenticate(
      'google',
      {
        failureRedirect: '/login',
      },
      (req, res) => {
        res.redirect('/');
      },
    ),
  );

  server.get('/logout', (req, res) => {
    // 將 req.user 以及 session 內的 user id 清空，轉導回 Login 頁面('/login')
    req.logout();
    res.redirect('/login');
  });
}

module.exports = setupGoogle;
```

---

#### User.publicFields 及 User.signInOrSignUp 函式

TODO: Check code

在我們的 `setupGoogle` 函式內我們在認證成功時會呼叫 `verify` 函式。`verify` 會從 Google 伺服器取得所有需要的資訊，例如 `googleToken` 及 `profile`。在 `verify` 函式內我們呼叫 `User.signInOrSignUp`，這會回傳一個新增的使用者或是一個既有的使用者。我們另外還會在 `passport.deserializeUser` 內呼叫 `User.publicFields`。但是我們目前還沒有定義 `User.publicFields` 及 `User.signInOrSignUp`。我們這個小節就來做這件事。

我們在第二章是透過定義 schema 並且將 schema 當作參數來呼叫 `mongoose.model` 來建立我們的 `User` 資料模型。不過我們目前還沒有針對我們的 `User` 資料模型定義任何靜態函式（static method）。一般來說，所有的資料模型都會有對應的一組新增 ∕ 讀取 ∕ 更新 ∕ 刪除（create/read/update/delete, CRUD）的靜態函式。這樣才能操作我們 MongoDB 內的 `User` 文件。

使用 Mongoose 的時候，我們是透過定義一個包含靜態函式的新的 `class` 然後呼叫 `mongoSchema.loadClass` 函式來提供資料模型所需要的靜態函式。

在我們的專案，我們會在 `/server/models/User.js` 內定義一個 `UserClass` 並且在呼叫 `mongoose.model('User', mongoSchema)` 之前呼叫 `mongoSchema.loadClass(UserClass)` 來綁定靜態函式：

```JavaScript
mongoSchema.loadClass(UserClass);

const User = mongoose.model('User', mongoSchema);

module.exports = User;
```

我們看一下 Mongoose 的官方文件上的範例：

[https://mongoosejs.com/docs/advanced_schemas.html#creating-from-es6-classes-using-loadclass](https://mongoosejs.com/docs/advanced_schemas.html#creating-from-es6-classes-using-loadclass)

```JavaScript
class PersonClass {
  static findByFullName() {
    // 省略
  }
}

schema.loadClass(PersonClass)
```

因此我們的 `UserClass` 大概框架如下：

```JavaScript
class UserClass {
  static publicFields() {
    // 取得 publicField
  }

  // 因為這裡面會呼叫 Mongoose API, 所以用 async
  static async signInOrSignUp() {
    // 取得已存在的 user, 如果不存在, 新增一位
  }
}
```

`User.publicFields` 函式將會提供可以不危害資安，可以放心提供給瀏覽器的使用者屬性（`id`、`displayName`、`email`、`slug`）。為了要正確的根據權限渲染頁面，我們需要告訴瀏覽器，該使用者是否具有 Admin 權限。另外，我們也想要顯示使用者的頭像，因此除了上述的四個屬性之外，另外有 `isAdmin` 及 `avatarUrl`：

```JavaScript
static publicFields() {
    // 取得 publicField
    return ['id', 'displayName', 'email', 'avatarUrl', 'slug', 'isAdmin'];
}
```

可以注意下，`User.publicFields` 不需要另外呼叫 Mongoose API，這是因為在 `passport.deserializeUser` 是透過 `User.findById` 使用 `User.publicFields`，因此已經會透過 `User.findById()` 取得 MongoDB 內的文件資訊：

```JavaScript
passport.deserializeUser((id, done) => {
    User.findById(id, User.publicFields(), (err, user) => {
      done(err, user);
    });
});
```

`User.signInOrSignUp` 就稍微複雜一些，因為 `User.signInOrSignUp` 的實際功用有以下：

1. 如果使用者存在，透過 `findOne` Mongoose API 找出此使用者，並且透過 `updateOne` Mongoose API 來將 token 更新為 `googleToken` 內的值

2. 如果使用者不存在，產出一個使用者 `displayName` 的對應 `slug` 值，再透過 `create` Mongoose API 來新增此使用者

```JavaScript
class UserClass {
  static publicFields() {
    // 取得 publicField
    return ['id', 'displayName', 'email', 'avatarUrl', 'slug', 'isAdmin'];
  }

  static async signInOrSignUp({ googleId, email, googleToken, displayName, avatarUrl }) {
    const user = await this.findOne({ googleId }).select(UserClass.publicFields().join(' '));

    if (user) {
      // 1. user 存在，update token
    }
    // 2. user 不存在，產出 slug 並新增 MongoDB 文件
  }
}
```

`select` 這個 Mongoose API 會指定要回傳 MongoDB 該文件的哪幾個欄位：

[http://mongoosejs.com/docs/api.html#query_Query-select](http://mongoosejs.com/docs/api.html#query_Query-select)

因此我們將 `User.publicFields().join(' ')` 作為參數，指定要回傳我們指定的欄位。

JavaScript 的 `join` 函式會將陣列內的各元素組成一個字串，下面是個範例：

```JavaScript
const elements = ['Fire', 'Wind', 'Rain'];

console.log(elements.join(' '));

// console: Fire Wind Rain
```

[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join)

我們來看一下 `User.signInOrSignUp` 內的兩塊主邏輯：

1. 如果 MongoDB 內存在這個使用者，我們要更新 token（更新為 `googleToken` 內的值）。

   這是因為，token 使用使用期限的，既然使用者現在嘗試登入，我們將 token 換為新的值也代表使用期限會重新計算。

   `googleToken` 基本上是個具有 `access_token`、`refresh_token`及其他屬性的物件：

   ```JavaScript
   "googleToken": {
     "access_token": "blah blah blah",
     "refresh_token": "sometokenvalue",
     "token_type": "Bearer",
     "expiry_date": 1510363003035
   }
   ```

   Google 針對不同 token 有不同的有效期限邏輯，以 `refresh_token` 為例：

   - 使用者三個月以上沒有使用
   - 使用者修改了他的 Google 帳號密碼
   - 使用者手動取消權限

   我們透過檢查 `googleToken` 內是否存有各 token 的值。假設有，我們就將該值以屬性的方式存放到 `modifier` 物件內。我們在呼叫 `updateOne` Mongoose API 時會將 `modifier` 作為參數：

   ```JavaScript
    const modifier = {};

    if (googleToken.accessToken) {
      modifier.access_token = googleToken.accessToken;
    }

    if (googleToken.refreshToken) {
      modifier.refresh_token = googleToken.refreshToken;
    }
   ```

   如果 `modifier` 是個空物件 - 例如 Google 沒有提供 token - 則我們就將使用者回傳而不更動資料庫內的 token 值：

   ```JavaScript
   // 要在檔案上方加上 lodash 的 import
   // const _ = require('lodash');
   if (_.isEmpty(modifier)) {
      return user;
   }
   ```

   我們可使用 `lodash` 套件的 `isEmpty` 函式來檢查 `modifier` 物件是否是空物件：

   [https://lodash.com/docs/4.17.4#isEmpty](https://lodash.com/docs/4.17.4#isEmpty)

   [https://lodash.com/docs](https://lodash.com/docs)

   如果 `modifier` 非空，我們則用 `modifier` 當參數更新使用者：

   ```JavaScript
   await this.updateOne({ googleId }, { $set: modifier });
   ```

   上述個步驟結合起來：

   ```JavaScript
   if (user) {
      // 1. user 存在，update token
      const modifier = {};

      if (googleToken.accessToken) {
        modifier.access_token = googleToken.accessToken;
      }

      if (googleToken.refreshToken) {
        modifier.refresh_token = googleToken.refreshToken;
      }

      if (_.isEmpty(modifier)) {
        return user;
      }

      await this.updateOne({ googleId }, { $set: modifier });

      return user;
   }
   ```

2. 如果 MongoDB 資料庫內並不存在對應的 `User` 文件，我們先產生一個 `slug`（這是使用下個小節會定義的 `generateSlug` 函式）：

   ```JavaScript
   const slug = await generateSlug(this, displayName);
   ```

   當我們取得 `slug` 後，我們會透過 `create` Mongoose API 來產生新的 `User` MongoDB 文件：

   ```JavaScript
   // 2. user 不存在，產出 slug 並新增 MongoDB 文件
   const slug = await generateSlug(this, displayName);
   const userCount = await this.find().countDocuments();

   const newUser = await this.create({
      createdAt: new Date(),
      googleId,
      email,
      googleToken,
      displayName,
      avatarUrl,
      slug,
      isAdmin: userCount === 0,
   });

   return _.pick(newUser, UserClass.publicFields());
   ```

   上面的 `this.find().countDocuments()` 會回傳一個集合內的文件數量：

   [https://mongoosejs.com/docs/api.html#model_Model.countDocuments](https://mongoosejs.com/docs/api.html#model_Model.countDocuments)

   如果我們的資料庫內完全沒有使用者，我們會將第一個登入的使用者設定為 Admin 角色。因此當 `userCount === 0` 時，我們會對應的將 `isAdmin: true`。

   最後回傳使用的 `pick` lodash 函式會*僅將*安全的欄位（我們前面定義的 `UserClass.publicFields`） 給取出，建立這個物件回傳。

結合上述的兩個步驟，`UserClass` 便如下：

```JavaScript
class UserClass {
  static publicFields() {
    // 取得 publicField
    return ['id', 'displayName', 'email', 'avatarUrl', 'slug', 'isAdmin'];
  }

  static async signInOrSignUp({ googleId, email, googleToken, displayName, avatarUrl }) {
    const user = await this.findOne({ googleId }).select(UserClass.publicFields().join(' '));

    if (user) {
      // 1. user 存在，update token
      const modifier = {};

      if (googleToken.accessToken) {
        modifier.access_token = googleToken.accessToken;
      }

      if (googleToken.refreshToken) {
        modifier.refresh_token = googleToken.refreshToken;
      }

      if (_.isEmpty(modifier)) {
        return user;
      }

      await this.updateOne({ googleId }, { $set: modifier });

      return user;
    }
    // 2. user 不存在，產出 slug 並新增 MongoDB 文件
    const slug = await generateSlug(this, displayName);
    const userCount = await this.find().countDocuments();

    const newUser = await this.create({
      createdAt: new Date(),
      googleId,
      email,
      googleToken,
      displayName,
      avatarUrl,
      slug,
      isAdmin: userCount === 0,
    });

    return _.pick(newUser, UserClass.publicFields());
  }
}
```

我們更新完的 `server/models/User.js` 更新如下：

```JavaScript
const mongoose = require('mongoose');
const _ = require('lodash');
// generateSlug 我們下個小節就會定義
const generateSlug = require('../utils/slugify');

const { Schema } = mongoose;

const mongoSchema = new Schema({
  googleId: {
    type: String,
    required: true,
    unique: true,
  },
  googleToken: {
    access_token: String,
    refresh_token: String,
    token_type: String,
    expiry_date: Number,
  },
  slug: {
    type: String,
    required: true,
    unique: true,
  },
  createdAt: {
    type: Date,
    required: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  isAdmin: {
    type: Boolean,
    default: false,
  },
  displayName: String,
  avatarUrl: String,
});

class UserClass {
  static publicFields() {
    // 取得 publicField
    return ['id', 'displayName', 'email', 'avatarUrl', 'slug', 'isAdmin'];
  }

  static async signInOrSignUp({ googleId, email, googleToken, displayName, avatarUrl }) {
    const user = await this.findOne({ googleId }).select(UserClass.publicFields().join(' '));

    if (user) {
      // 1. user 存在，update token
      const modifier = {};

      if (googleToken.accessToken) {
        modifier.access_token = googleToken.accessToken;
      }

      if (googleToken.refreshToken) {
        modifier.refresh_token = googleToken.refreshToken;
      }

      if (_.isEmpty(modifier)) {
        return user;
      }

      await this.updateOne({ googleId }, { $set: modifier });

      return user;
    }
    // 2. user 不存在，產出 slug 並新增 MongoDB 文件
    const slug = await generateSlug(this, displayName);
    const userCount = await this.find().countDocuments();

    const newUser = await this.create({
      createdAt: new Date(),
      googleId,
      email,
      googleToken,
      displayName,
      avatarUrl,
      slug,
      isAdmin: userCount === 0,
    });

    return _.pick(newUser, UserClass.publicFields());
  }
}

mongoSchema.loadClass(UserClass);

const User = mongoose.model('User', mongoSchema);

module.exports = User;

```

我們離完成又更近一步了，剩下的步驟有：

- 定義 `generateSlug` 函式
- 將我們網頁應用程式與 Google Cloud 進行連結設定，取得 `GOOGLE_CLIENTID` 以及 `GOOGLE_CLIENTSECRET`
- 測試驗證！

---

#### generateSlug 函式

上一個小節，我們定義了 `UserClass` 兩個靜態函式，這也連帶會變成 `User` 資料模型的靜態函式。其中 `User.signInOrSignUp` 會呼叫 `server/utils/slugify.js` 引入的 `generateSlug` 函式。`generateSlug` 函式是個工具型的函式 - 因此我們將這個檔案放在 `server/utils/*` 目錄下。我們雖然使用了 `generateSlug` 函式，但是我們還沒實際定義實作這個函式，這小節我們就來做這件事。

我們先來看一下如何從 `displayName` 產生*非獨特*的 `slug` 值。非獨特在這邊的意思是，我們不會去檢查是否有重複一樣的 `slug`。舉例來說，如果有兩個使用者，他們的 `displayName` 都是 `John`，則兩個使用者的 `slug` 都會是 `john`。如果存進我們的 MongoDB 內，則 `users` 集合內會有兩筆 MongoDB 文件，其 `slug` 欄位的值是一樣的（都是 `john`）。如果我們要獨特的 `slug` 欄位，我們應該要對其中一筆的 `slug` 加上 `-1`，得到 `john` 及 `john-1` 兩個 slug，以維持 `users` 集合內各文件的 `slug` 欄位都是唯一的。

非獨特的 `slug` 很簡單就可以得到，我們使用 `lodash` 套件的 `kebabCase` 函式。`kebabCase` 函式作用如下：

```JavaScript
_.kebabCase('Foo Bar');
// => 'foo-bar'

_.kebabCase('fooBar);
// => 'foo-bar'

_.kebabCase('__FOO_BAR__');
// => 'foo-bar'
```

可以看到 `kebabCase` 會將字變小寫，然後將字與字之間以 `-` 給串起來（kebab 正是串燒的英文），這正是 `slugify` 所需，不過當然，這並沒有檢查結果的獨特性。

官方文件可以參考：

[https://devdocs.io/lodash~4/index#kebabCase](https://devdocs.io/lodash~4/index#kebabCase)

所以我們先定義 `slugify` 這個函式：

```JavaScript
const _ = require('lodash');

const slugify = (text) => _.kebabCase(text);
```

所以我們現在來看看怎麼在 `generateSlug` 函式加上獨特性的檢查。`generateSlug` 這個函式檢查獨特性的方式必須要與資料模型本身沒有相依性，換言之，任何資料模型（`User` 以及我們未來要加上的 `Book` 與 `Chapter` 模型）都要能夠使用。此函式會接受 `Model` 及 `name` 為參數（另外還會有個預設值為 `{}` 的 `filter` 參數，本章還不會用到，不用太在意它），並回傳一個獨特的 `slug` 值。

第一步我們先簡單的取得未必獨特的 slug：

```JavaScript
const origSlug = slugify(name);
```

接著我們使用 `Model.findOne` Mongoose API 來檢查資料庫內是否有 user 的 `slug` 欄位與 `origSlug` 一樣的文件（也就是這個 slug 已經有人用了）：

```JavaScript
const user = await Model.findOne({ slug: origSlug, ...filter }, 'id');
```

如果沒有撞名的 user，則此 `slug` 是獨特的，我們就將之回傳：

```JavaScript
if (!user) {
  return origSlug;
}
```

如果有撞名的 user（也就是 `user` 有值），則目前的 `origSlug` 已經被使用了。
這時我們會呼叫一個 `createUniqueSlug` 函式，此函式會將 `origSlug` 嘗試接上
一個 `-數字` 例如 `-1`，來讓 slug 變成獨特的值。

綜合以上，你的 `server/utils/slugify.js` 現在應該會如下：

```JavaScript
const _ = require('lodash');

const slugify = (text) => _.kebabCase(text);

async function generateSlug(Model, name, filter = {}) {
  const origSlug = slugify(name);

  const user = await Model.findOne({ slug: origSlug, ...filter }, 'id');

  if (!user) {
    return origSlug;
  }

  return createUniqueSlug(Model, origSlug, 1);
}

module.exports = generateSlug;
```

剩下的就只有定義 `createUniqueSlug` 了，非常類似於 `generateSlug`：

1. 先將原始 `slug` 加上一個數字（第三個參數），變成 `${slug}-${count}`
2. 跟 `generateSlug` 一樣，檢查 `slug`（但是現在 `slug` 的值是 `${slug}-${count}`）是否被佔用
3. 假設沒被佔用，代表 `${slug}-${count}` 是獨特的，回傳此值
4. 假設被佔用，則將 `count` 加一，遞迴從頭開始

函式實作如下：

```JavaScript
async function createUniqueSlug(Model, slug, count) {
  const user = await Model.findOne({ slug: `${slug}-${count}` }, 'id');

  if (!user) {
    return `${slug}-${count}`;
  }

  return createUniqueSlug(Model, slug, count + 1);
}
```

我們實際演練看看 `generateSlug` 是如何運行。假如有個使用者，他的 `displayName` 值是 `Jack Frost`，他嘗試在我們的網頁應用程式註冊。如果我們的資料庫已經存在一個 `slug` 值為 `jack-frost` 的使用者，而且沒有存在 `jack-frost-1` 的使用者，則 `generateSlug` 會回傳 `jack-frost-1`。如果 `jack-frost` 及 `jack-frost-1` 都有使用者佔用的時候，`generateSlug` 則會回傳 `jack-frost-2`。

我們的 `server/utils/slugify.js` 如下：

```JavaScript
const _ = require('lodash');

const slugify = (text) => _.kebabCase(text);

async function createUniqueSlug(Model, slug, count) {
  const user = await Model.findOne({ slug: `${slug}-${count}` }, 'id');

  if (!user) {
    return `${slug}-${count}`;
  }

  return createUniqueSlug(Model, slug, count + 1);
}

async function generateSlug(Model, name, filter = {}) {
  const origSlug = slugify(name);

  const user = await Model.findOne({ slug: origSlug, ...filter }, 'id');

  if (!user) {
    return origSlug;
  }

  return createUniqueSlug(Model, origSlug, 1);
}

module.exports = generateSlug;

```

大功告成，`generateSlug` 函式會回傳獨特的 `slug` 值了。

下一小節，我們在 Google Cloud Platform 上建立一個對應我們專案的網頁應用程式，這樣就可以測試我們整合 Google OAuth API 的成果了。

---

#### GCP（Google Cloud Platform）設定與測試

真的保證，我們快完成了。

我們之前已經在 `server/server.js` 內引用了 `setupGoogle` 並呼叫它。現在總算將這個函式定義實作完畢。

我們現在可以將本來透過 Express 路徑（`/`） hard code 搜尋 `user` 的部分可以刪除了，因為我們現在會透過 `setupGoogle` 來進行認證：

```JavaScript
// 這段不用了，因為 setupGoogle 會處理了
server.get('/', async (req, res) => {
  req.session.foo = 'bar';
  const user = await User.findOne({ slug: 'pheno-author' });
  req.user = user;
  app.render(req, res, '/');
});
```

對應的我們也不需要 `User` 的引入了：

```JavaScript
// 這行也不用了，server/server.js 沒在用了
const User = require('./models/User');
```

注意一下 `server.use(session(sess))` 需要在 `setupGoogle({ server, ROOT_URL })` 之前。`session` 中介層需要在 `passport.session` 中介層（這位於 `setupGoogle`）之前才能確保 login session 的正確運作，可以參考：

[http://www.passportjs.org/docs/configure](http://www.passportjs.org/docs/configure)

---

總算，我們開發告一段落，現在只剩下將 Google API 所需要的金鑰並將之加到 `.env` 檔案內。

到 Google Cloud Platform（如果沒帳號的話，建立一個），去設定你的網頁應用程式的 OAuth 認證資料。下方是所有過程的正式官方教學：

[https://developers.google.com/identity/sign-in/web/devconsole-project](https://developers.google.com/identity/sign-in/web/devconsole-project)

由於官方文件相當清楚，我們不重複，就照著做，取得 OAuth 認證資料（`GOOGLE_CLIENTID` 及 `GOOGLE_CLIENTSECRET`）。唯一要注意的是，在設定時，`Authorized JavaScript origins` 是 `http://localhost:8000`，而 `Authorized redirect URIs` 是 `http://localhost:8000/oauth2callback`（這組是用在我們當地本機使用，以後要正式發布時要在補上正式的 URL）。

得到 `GOOGLE_CLIENTID` 及 `GOOGLE_CLIENTSECRET` 後，將它們加到 `.env` 檔案內。

我們可以測試了！能撐到這邊的，我要為你鼓掌。`yarn dev` 並造訪 `http://localhost:8000/login`。

登入頁面：

![alt login](./3-images/3-login.png)

點擊 `使用 Google 登入`，會出現選擇 Google 帳號的選單：

![alt account selection](./3-images/3-account-selection.png)

接者會轉導至已登入的首頁（可以看到對的頭像及 email）：

![alt logged in](./3-images/3-logged-in.png)

點擊你的頭像，並選擇 `登出` 選項，你會登出並且回到登入頁面：

![alt log out](./3-images/3-log-out.png)

到 Mongo Atlas 看 `bookstore.users` 集合，如果你完全照書上的操作至今，你會看到兩筆 `user` 文件：

- 第一筆是我們前兩章的時候手動建立的
- 第二筆是我們剛剛（透過 `User.signInOrSignUp`）所建立的

![alt mongo users](./3-images/3-mongo-users.png)

可以注意到第二筆的 `isAdmin` 是 `false`（因為我們的邏輯是只有把第一個建立的使用者設定為 Admin）。

我們將這兩筆 user 刪除（手動），然後重新在網頁上登入一次：

![alt mongo single user](./3-images/3-mongo-admin.png)

可以看到這次 `isAdmin` 變成 `true` 了。

你可以造訪 [https://myaccount.google.com/permissions](https://myaccount.google.com/permissions) 查看有哪些應用程式透過 Google API 取得你的 Google profile 資訊。

我們最後測試永久登入的功能。將頁籤關掉，開一個新的頁籤，並造訪 `http://localhost:8000` - 仍然是登入狀態。看一下 `Developer tools > Application > Cookies > http://localhost:8000`：

![alt login cookie](./3-images/3-login-cookie.png)

選擇 `pheno-bookstore`（你有可能用了別的名稱）cookie，並刪除它。重整頁籤 - **你被登出了！** 如果你有跟著書上內容在學習，這個行為應該是可預期的：

沒有 cookie => 無法找到 session => 無法找到 user id => 無法組出 `req.user` => `Index` 頁面沒有得到 user 資訊 => 判斷為未登入

如果你的測試結果跟上述的都一樣，恭喜你完成了 Google OAuth API 整合。

TODO: Google OAuth API Flow

我們下一章（第四章）將會看一下如何使用 Jest 套件測試程式，並且讓我們頁面可以呈現提示資訊給我們的使用者。

---
TODO: fix up code
這章到這裡是尾聲，你的程式現在應該長得像 `book/3-end` 的內容。

可以比較一下，修改有問題的地方。

如果有發現任何 bug、錯字或是解釋不清楚的部分，歡迎透過 pheno_the_best@yahoo.com.tw 告知。

如果你覺得看了這本書有收穫，也歡迎給我們一些書評。也一樣是歡迎將書評寄到 pheno_the_best@yahoo.com.tw，謝謝！

---