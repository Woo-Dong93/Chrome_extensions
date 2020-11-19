# 리액트 + 타입스크립트 = 크롬 확장프로그램

- 이 프로젝트는 **리액트** + **타입스크립트**를 직접 `웹팩`과 `바벨`을 활용하여 초기 셋팅을 할 수 있습니다.
- 크롬 확장프로그램까지 연동 및 배포를 체험해볼 수 있습니다.
- 이해를 돕기 위해 최소한의 라이브러리만 사용하였습니다.



### 1. 초기 셋팅

- react

```
npm i react react-dom react-frame-component prop-types
```

		- react-frame-component, prop-types : iframe을 활용하기 위한 라이브러리 입니다.
		- 직접 클라이언트 화면에 태그를  추가하는 방식이다보니 CSS영향을 받게 되어서 그 부분을 개선하고자 iframe을 활용했습니다.



- babel

```
npm i -D @babel/cli @babel/core @babel/preset-env @babel/preset-react babel-loader
```



- webpack

```
npm i -D webpack webpack-cli webpack-dev-server html-webpack-plugin
```

```
npm i -D style-loader css-loader copy-webpack-plugin
```

- html-webpack-plugin : html 파일을 공백없이 번들링 해줍니다.
- style-loader, css-loader : .css 파일을 import 할 수 있으며 번들링 해줍니다.
- copy-webpack-plugin : 그 외에 기타파일을 웹팩 결과물과 함께 그대로 가지고 올 수 있습니다.



- typescript

```
npm i -D typescript ts-loader
```

```
npm i @types/react @types/react-dom @types/react-frame-component
```

```
npm i @types/chrome
```



- .babelrc 파일 설정

```json
{
    "presets": ["@babel/preset-env", "@babel/preset-react"]
}
```



- tsconfig.json
  - 자동생성 이용하기 ( **npx typescript --init** )

```json
{
  "compilerOptions": {
    "jsx": "react",
    "target": "es5",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```



- webpack.config.js

```json
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const CopyWebpackPlugin = require("copy-webpack-plugin");
const mode = process.env.NODE_ENV || 'development';

module.exports = {
  mode,
  entry: {
    main: './src/index.tsx',
  },
  output: {
    path: path.resolve('./dist'),
    filename: '[name].js',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
        exclude: /node_module/,
      },
      { test: /\.tsx?$/, loader: "ts-loader" },
      {
        test: /\.css$/,
        use: ["style-loader", "css-loader"],
      }    
    ],
  },
  resolve: {
    extensions: [".ts", ".tsx", ".js"],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
    }),
    new CleanWebpackPlugin(),
    new CopyWebpackPlugin({
      patterns:[
        { from: './public/background.js', to: './' },
        { from: './public/manifest.json', to: './' },
        { from: './public/icon.png', to: './' },
      ]
    })
  ],
  optimization: {
  },
  devServer: {
    contentBase: path.join(__dirname, 'dist'),
    publicPath: '/',
    host: 'localhost',
    overlay: true,
    port: 8080,
    hot: true,
    stats: 'errors-only',
    historyApiFallback: true,
  },
};

```



### 2. 타입스크립트로 리액트 컴포넌트 생성

- index.tsx

```javascript
import * as React from "react"
import * as ReactDOM from "react-dom"
import App from './components/App';
import "../public/root.css";

const app = document.createElement('div');
app.id = "my-extension";
document.body.appendChild(app);
app.style.display = "none";

chrome.runtime.onMessage.addListener(
   function(request, sender, sendResponse) {
      if( request.message === "click") {
        toggle();
      }
   }
);

function toggle(){
   if(app.style.display === "none"){
     app.style.display = "block";
   }else{
     app.style.display = "none";
   }
}

ReactDOM.render(<App name='우동이' />, app);
```



- App.tsx

```typescript
import * as React from 'react';
import Frame, { FrameContextConsumer }from 'react-frame-component';
import Content from './Content';
interface AppProps {
  name: string;
}

const App: React.FC<AppProps> = props => {
  const { name } = props;
  return (
    <Frame> 
      <FrameContextConsumer>
      {
        ({document, window}) => {
          return <Content name={name}/> 
        }
      }
      </FrameContextConsumer>
    </Frame>
  );
}

export default App;
```



- Context.tsx

```typescript
import * as React from 'react';

interface ContentProps {
  name: string;
}

const Content: React.FC<ContentProps> = props => {
  const { name } = props;
  const style = {
    textAlign: 'center' as const
  }
  return (
    <div style={style}>
      <h1>{name}의 확장 프로그램</h1>
      <span>확장 프로그램을 연동해보는 프로젝트 입니다.</span>
    </div>
  );
}

export default Content;
```



### 3. 크롬 확장프로그램 연동해보기

- public 폴더를 만들고 여기에 크롬 확장프로그램에 관련된 파일들을 모아둡니다.
- 웹팩 설정파일에서 이미 **CopyWebpackPlugin**을 활용하여 설정을 끝마쳐 번들링할 때 여기 있는 파일들을 가져갑니다.



- background.js
  - 크롬 확장프로그램 자체의 자바스크립 파일입니다.

```javascript
// 사용자가 확장프로그램 아이콘을 클릭할 때에
chrome.browserAction.onClicked.addListener(function(tab) {
    // 현재 사용자가 보고 있는 탭( 웹사이트 )의 정보를 가져와서
   chrome.tabs.query({active: true, currentWindow:true},function(tabs) {
        var activeTab = tabs[0];
       // 가져온 탭 ( 웹사이트 )에게 메세지( click )를 전송합니다.
       // index.tsx 파일을 보시면 이 메세지를 수신하여 toggle()를 이용하여 해당 컨텐츠를 보여주거나 제거합니다.
        chrome.tabs.sendMessage(activeTab.id, {"message": "click"});
   });
});
```



- manifest.json
  - 크롬 확장프로그램 셋팅 파일입니다.
  - **name**, **version** : 확장프로그램의 정보를 입력하는 곳입니다.
  - **background** : background.js파일을 등록하는 곳입니다.
  - **default_popup** : 확장프로그램을 클릭시 나오는 모달의 파일을 입력하는 곳입니다. 우리는 팝업 모달창을 사용하지 않고 직접 웹사이트에 접근하여 **toggle**형식으로 해당 컨텐츠를 보여주고 있습니다.
  - **default_icon** : 확장프로그램의 아이콘 파일을 입력하는 곳입니다.
  - **content_scripts** : 사용자가 보고 있는 페이지에 대한 셋팅입니다.
    - **matches** : 여기에 입력된 주소만 확장프로그램이 작동하게 됩니다.
    - **css** : css파일을 입력하면 해당 css파일이 현재 페이지에 적용됩니다.
    - **js** : js파일을 입력하면 해당 js파일이 현재 페이지에 적용됩니다. 이 부분은 웹팩으로 번들링 된 결과물을 등록해줍니다.

```json
{
  "name": "My Extension",
  "version": "1.1",
  "manifest_version": 2,
  "background": {
    "scripts": ["background.js"]
  },
  "browser_action": {
    "default_popup": "",
    "default_icon": "icon.png"
  },
  "content_scripts" : [
    {
      "matches": [ "<all_urls>" ],
      "css": [],
      "js": ["/main.js"]
    }
  ]
}
```



- root.css

```css
#my-extension{
  width: 100%;
  height: 350px;
  position: fixed;
  bottom: 0px;
  left: 0px;
  z-index: 10000;
  background-color: green;
  border: 1px solid red;
}

#my-extension iframe {
  width: 100%;
  height: 100%;
  border: none;
}
```



### 4. 크롬에 배포해보자!

- 배포전에 크롬브라우저에서 설정 => 도구 더보기 => 확장프로그램 => 개발자 모드 On으로 바꾼 뒤에 압축해제된 **[확장 프로그램을 로드합니다]**를 클릭하여 프로젝트를 로드해서 배포전에 테스트할 수 있습니다.

- https://chrome.google.com/webstore/category/extensions?hl=ko
- 개발자 대시보기 => 로그인 => 수수료 지불 => 압축한 프로젝트를 업로드할 수 있습니다.
- 로고 및 카테고리 등록 가능합니다.
- 언어 등록 가능합니다.
- 내용이 수정되면 manifest.json 에서 버전을 변경하고 올리면 됩니다.



![image](https://user-images.githubusercontent.com/52816790/99692955-3245d900-2ace-11eb-9eb4-adb3c7ee1cd3.png)