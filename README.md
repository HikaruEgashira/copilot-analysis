# 花了大半个月，我终于逆向分析了Github Copilot

## 背景

Github Copilotは、機械学習に基づくコード補完ツールです。GitHubから大量のコードをトレーニングデータとして使用し、OpenAIの言語モデルを使用してコードを生成します。Copilotは、ユーザーのコーディング習慣を学習し、コンテキストから正しいコードスニペットを推測することができます。

実際の使用では、大部分のヒントが非常に役立つことがわかりました。ユーザーの意図を比較的正確に推測し、プロジェクトの他のファイルのコンテキストに基づいて推論することさえできます。ここがどうやってやっているのか、このVSCodeプラグインの詳細な実装を探ってみました。

## 准备工作

Copilotがオープンソースでないため、いくつかの逆向きの準備をする必要があります。

まず、VSCodeプラグインのインストールディレクトリを見つけ、`extension.js`を取得します。

![image](https://files.mdnice.com/user/13429/8966f2a1-c7ee-457d-a233-b53d18d4f80f.png)

macでプラグインディレクトリは`~/.vscode`の下にあります。圧縮と難読化されたファイルを取得できます：

![image](https://files.mdnice.com/user/13429/1c2b9452-87f1-45d3-8ba7-4b2fc1a69a56.png)

### 1. 分割`webpack_modules`

webpackの圧縮混合されたjsに対して、まずwebpackのbundleを識別して、単一ファイルに分割して分析するために必要です。

圧縮されたコードには多くの不確実性が存在するため、最初に正規表現で抽出しようとしましたが、どのような状況でも正しく抽出されませんでした。最も簡単な方法は、ASTを使用して抽出することです。

まず、babel-parserを使用してソースコードをASTに解析します。

```jsx
const ast = parser.parse(source);
```

そして、babel-traverseを通してASTを一回りし、modulesの変数を見つけて、中身を取り出す：

```jsx
function parseModules() {
  traverse(ast, {
    enter(path) {
      if (
        path.node.type === "VariableDeclarator" &&
        path.node.id.name === "__webpack_modules__"
      ) {
        const modules = path.node.init.properties;
        for (const module of modules) {
          const moduleId = module.key.value;
          const moduleAst = module.value;
          const moduleSource = generate(moduleAst).code;

          try {
            const ast = transformRequire(prettier(clearfyParams(moduleId, moduleSource)));

            const mainBody = ast.program.body[0].expression.body.body;
            const moduleCode = generate(types.Program(mainBody)).code;
            fs.writeFileSync(
              "./prettier/modules/" + moduleId + ".js",
              moduleCode,
              "utf8"
            );
          } catch (e) {
            console.log(e);
          }
        }
      }
    },
  });
}

```

最後に、処理したastをbabel-generatorとbabel-typesを使って新しいastに変換し、ファイルに書き込みます。
これで、モジュールIDで名前を付けた独立bundleを得ることができます。
このバージョンでは、解析されたcopilotのbundleの数はすでに多くなっており、752個に達しています。

![image](https://files.mdnice.com/user/13429/3a674c5a-0b99-4706-a5cb-b58316767cf1.png)

### 2. 识别模块依赖

私たちが解析したバンドル、第一層の関数は大体こんな感じです：

![image](https://files.mdnice.com/user/13429/8ab939a1-0421-4870-ac58-76807ffe91e9.png)

webpackにとって、これらの引数は固定値であり、それぞれ`module`、`exports`、`require`です。したがって、まずこれらの引数を識別し、スコープを置き換える必要があります。そうでないと、モジュール間の依存関係を確認できません。

```jsx
function clearfyParams(moduleId, moduleSource) {
  if (moduleSource.trim().startsWith("function")) {
    // change `function(e, t, n) {` to `(e, t, n) => {`
    moduleSource = moduleSource.replace("function", "");
    moduleSource = moduleSource.replace(")", ") =>");
  }

  const moduleAst = parser.parse(moduleSource);
  let flag = false;

  traverse(moduleAst, {
    ArrowFunctionExpression(path) {
      if (flag) return;
      const params = path.node.params;
      params.forEach((param) => {
        if (param.name === "e" || param.name === "t" || param.name === "n") {
          path.scope.rename(
            param.name,
            {
              e: "module",
              t: "exports",
              n: "require",
            }[param.name]
          );
        }
      });
      flag = true;
    },
  });
  return moduleAst;
}
```

これで、requireとexportsを持つコードを得ることができました：

```jsx
var r = require(12781).Stream;
var i = require(73837);
function o() {
  this.source = null;
  this.dataSize = 0;
  this.maxDataSize = 1048576;
  this.pauseStream = !0;
  this._maxDataSizeExceeded = !1;
  this._released = !1;
  this._bufferedEvents = [];
}
module.exports = o;
```

### 3. 优化压缩后的语法

JSコードは圧縮されると、カンマ演算子、ショートサーキット記法、三項演算子、カッコクロージャーなどが大量に生成され、読み取りが非常に妨げられますが、ここでは以下を参考にしました
https://github.com/thakkarparth007/copilot-explorer
このプロジェクトが行った逆向きの作業について、文法をいくつかの処理を行いました：

```jsx
function prettier(ast) {
  const moduleTransformer = {
    // e.g., `(0, r.getConfig)(e, r.ConfigKey.DebugOverrideProxyUrl);`
    // gets transformed to r.getConfig(e, r.ConfigKey.DebugOverrideProxyUrl);
    CallExpression(path) {
      if (path.node.callee.type != "SequenceExpression") {
        return;
      }
      if (
        path.node.callee.expressions.length == 2 &&
        path.node.callee.expressions[0].type == "NumericLiteral"
      ) {
        path.node.callee = path.node.callee.expressions[1];
      }
    },
    ExpressionStatement(path) {
      if (path.node.expression.type == "SequenceExpression") {
        const exprs = path.node.expression.expressions;
        let exprStmts = exprs.map((e) => {
          return types.expressionStatement(e);
        });
        path.replaceWithMultiple(exprStmts);
        return;
      }
      if (path.node.expression.type == "AssignmentExpression") {
        // handle cases like: `a = (expr1, expr2, expr3)`
        // convert to: `expr1; expr2; a = expr3;`
        if (path.node.expression.right.type == "SequenceExpression") {
          const exprs = path.node.expression.right.expressions;
          let exprStmts = exprs.map((e) => {
            return types.expressionStatement(e);
          });
          let lastExpr = exprStmts.pop();
          path.node.expression.right = lastExpr.expression;
          exprStmts.push(path.node);
          path.replaceWithMultiple(exprStmts);
          return;
        }

        // handle cases like: `exports.GoodExplainableName = a;` where `a` is a function or a class
        // rename `a` to `GoodExplainableName` everywhere in the module
        if (
          path.node.expression.left.type == "MemberExpression" &&
          path.node.expression.left.object.type == "Identifier" &&
          path.node.expression.left.object.name == "exports" &&
          path.node.expression.left.property.type == "Identifier" &&
          path.node.expression.left.property.name != "default" &&
          path.node.expression.right.type == "Identifier" &&
          path.node.expression.right.name.length == 1
        ) {
          path.scope.rename(
            path.node.expression.right.name,
            path.node.expression.left.property.name
          );
          return;
        }
      }
      if (path.node.expression.type == "ConditionalExpression") {
        // handle cases like: `<test> ? c : d;`
        // convert to: `if (<test>) { c; } else { d; }`
        const test = path.node.expression.test;
        const consequent = path.node.expression.consequent;
        const alternate = path.node.expression.alternate;

        const ifStmt = types.ifStatement(
          test,
          types.blockStatement([types.expressionStatement(consequent)]),
          types.blockStatement([types.expressionStatement(alternate)])
        );
        path.replaceWith(ifStmt);
        return;
      }
      if (path.node.expression.type == "LogicalExpression") {
        // handle cases like: `a && b;`
        // convert to: `if (a) { b; }`
        const test = path.node.expression.left;
        const consequent = path.node.expression.right;

        const ifStmt = types.ifStatement(
          test,
          types.blockStatement([types.expressionStatement(consequent)]),
          null
        );
        path.replaceWith(ifStmt);
        return;
      }
    },
    IfStatement(path) {
      if (!path.node.test || path.node.test.type != "SequenceExpression") {
        return;
      }
      const exprs = path.node.test.expressions;
      let exprStmts = exprs.map((e) => {
        return types.expressionStatement(e);
      });
      let lastExpr = exprStmts.pop();
      path.node.test = lastExpr.expression;
      exprStmts.push(path.node);
      path.replaceWithMultiple(exprStmts);
    },
    ReturnStatement(path) {
      if (
        !path.node.argument ||
        path.node.argument.type != "SequenceExpression"
      ) {
        return;
      }
      const exprs = path.node.argument.expressions;
      let exprStmts = exprs.map((e) => {
        return types.expressionStatement(e);
      });
      let lastExpr = exprStmts.pop();
      let returnStmt = types.returnStatement(lastExpr.expression);
      exprStmts.push(returnStmt);
      path.replaceWithMultiple(exprStmts);
    },
    VariableDeclaration(path) {
      // change `const a = 1, b = 2;` to `const a = 1; const b = 2;`
      if (path.node.declarations.length > 1) {
        let newDecls = path.node.declarations.map((d) => {
          return types.variableDeclaration(path.node.kind, [d]);
        });
        path.replaceWithMultiple(newDecls);
      }
    },
  };
  traverse(ast, moduleTransformer);
  return ast;
}
```

### 4. require的模块id取名

圧縮したコードでは、requireによる依存関係にはモジュールIDしかなく、元のファイル名が失われています。
そのため、ここでは手動でマッピング（もちろんGPTを利用することもできます）を行い、異なるファイルの名前を推測し、マップファイルを保持し、AST内のモジュールIDを意味のあるモジュール名に置き換えます。

```jsx
function transformRequire(ast) {
  const moduleTransformer = {
    VariableDeclaration(path) {
        if (path.node.declarations[0].init && path.node.declarations[0].init.type === "CallExpression") {
            if (path.node.declarations[0].init.callee.name === "require") {
                const moduleId = path.node.declarations[0].init.arguments[0].value;
                if (NameMap[moduleId]) {
                    const { name, path: modulePath} = NameMap[moduleId];

                    path.node.declarations[0].init.arguments[0].value = '"'+modulePath+'"';
                    path.scope.rename(path.node.declarations[0].id.name, name);
                }
              }
        }
      
    },
  };
  traverse(ast, moduleTransformer);
  return ast;
}
```

これで、関連する逆アセンブルの準備は完了しました。

## 入口分析

前回、大量の逆向工作を行ったにもかかわらず、実際には逆向きのJSコードは非常に多くの仕事を行うことができるので、限られた精力の下で、我々はいくつかの文脈を手動で圧縮する変数を推測し、いくつかのコアファイルのコードを復元することができるようになりました。

入り口は、モジュールIDは91238であることが非常に容易である。

![image](https://files.mdnice.com/user/13429/3e853967-0df6-4443-b95c-c7e9594ed381.png)

一連の手動の最適化操作を経て、我々はこのエントリファイルの元の姿を大まかに復元することができます：

![image](https://files.mdnice.com/user/13429/e9823fac-23da-474c-86ce-296bd0e9cc55.png)

VSCodeのactive関数で、copilotは多くの初期化関連の作業を行い、各モジュールのサンプルをコンテキストに登録します。 以降、インスタンスはコンテキストから取得します。

私たちのコアはcopilotのコード補完能力を探求したいと思っていますが、入り口ファイルの詳細についてはここでは展開しません。

## コード補完入口ロジック

コードのヒントロジックは、`registerGhostText`で登録されています：

![image](https://files.mdnice.com/user/13429/7a5b9151-3050-4daa-a46c-be52ce524450.png)

vscodeでは、主に`InlineCompletionItemProvider`を通じてエディタ内でのコード補完能力を実現している。

実装のエントリロジックは、大まかに以下のように復元されます。

![image](https://files.mdnice.com/user/13429/fad4aa2b-ac0e-4fef-b2f9-881592a8aad6.png)

全体的には、かなり明確です。大まかに以下のことを行っています。

- ユーザーが`InlineSuggestEnable`を閉じたり、ドキュメントが処理ホワイトリスト内になく、ユーザーが入力をキャンセルしたりした場合は、すぐにreturnして、コードのヒントを表示しません。
- `getGhostText`メソッドを呼び出して、textsを取得します。これは、ユーザーに返されるコードのヒントテキストです。
- `completionsFromGhostTextResults`を呼び出して、最終的なcompletionsを取得します。この関数はかなり簡単で、テキストにいくつかのフォーマット処理を行っています。たとえば、タブスペースの問題を処理し、カーソルの現在の位置に基づいて、コードのヒントがエディタに表示される座標範囲を計算します。

## getGhostText核心ロジック

getGhostTextは、ヒントコードを取得する核心メソッドであり、コード全体が多いため、分割してみましょう：

### 1. プロンプトを抽出する

```jsx
const prompt = await extractprompt.extractPrompt(ctx, document, position);
```

promptを抽出するのはちょっと複雑な操作なので、次の小節で詳細に分析します。
### 2. 境界判断

```jsx
if ("copilotNotAvailable" === prompt.type) {
    exports.ghostTextLogger.debug(
      ctx,
      "Copilot not available, due to the .copilotignore settings"
    );
    return {
      type: "abortedBeforeIssued",
      reason: "Copilot not available due to the .copilotignore settings",
    };
  }
  if ("contextTooShort" === prompt.type) {
    exports.ghostTextLogger.debug(ctx, "Breaking, not enough context");
    return {
      type: "abortedBeforeIssued",
      reason: "Not enough context",
    };
  }
  if (token?.isCancellationRequested) {
    exports.ghostTextLogger.info(ctx, "Cancelled after extractPrompt");
    return {
      type: "abortedBeforeIssued",
      reason: "Cancelled after extractPrompt",
    };
  }
```

ここでの境界範囲は主に三つの場合がある：

- .copilotignoreに含まれるファイル
- コンテキストが少なすぎる
- ユーザーがキャンセルした

### 3. 2段階キャッシュ

copilot内部で2つのキャッシュ処理を行っています。 1段階キャッシュは、前回の`prefix`と`suffix`を保存します：

```jsx
function updateGlobalCacheKey(prefix, suffix, promptKey) {
  prefixCache = prefix;
  suffixCache = suffix;
  promptKeyCache = promptKey;
}
```

ここでの`promptKey`は、`prefix`と`suffix`の内容に基づいて計算されます。copilotがリクエストをバックエンドに送信する前に、このリクエストのprefixとsuffixが以前と同じであると判断した場合、キャッシュされた内容が読み取られます。

![image](https://files.mdnice.com/user/13429/91ea686c-7696-452f-a18b-927b96243cac.png)

続いて、一次キャッシュがミスした場合、copilotは二次キャッシュを計算し、現在のpromptがキャッシュ範囲にあるかどうかを計算します。

![image](https://files.mdnice.com/user/13429/d7ff3141-d12c-4b10-89f1-c217cbe32681.png)

ここで、copilotが採用しているキャッシュはLRUキャッシュポリシーであり、promptはデフォルトで100件キャッシュされます。

```jsx
exports.completionCache = new s.LRUCacheMap(100);
```

而keyForPrompt函数就是一个对prefix和suffix的hash：

```jsx
exports.keyForPrompt = function (e) {
  return r.SHA256(e.prefix + e.suffix).toString();
};
```

### 4. 真正発起要求

本当に後ろ向きにprompt要求を送る時に、copilotはまだ細かいことを二つしている：

1. `Debounce`遅延を設定する
2. `contexualFilterScore`の閾値に達するかどうかを判断する

まず、頻繁に後ろ向きに要求を送るのを避けるために、copilotはdebounceをして、モデルの計算がとても算力を消費するので、このシーンでは、debounceをする必要がある。copilotのこのdebounceも一般的ではない、私たちは実装の詳細を見てみましょう：

```jsx
exports.getDebounceLimit = async function (e, t) {
  let n;
  if ((await e.get(r.Features).debouncePredict()) && t.measurements.contextualFilterScore) {
    const e = t.measurements.contextualFilterScore;
    const r = .3475;
    const i = 7;
    n = 25 + 250 / (1 + Math.pow(e / r, i));
  } else n = await e.get(r.Features).debounceMs();
  return n > 0 ? n : 75;
};
```

copilotには予測スイッチがあり、現在の内容の関連性スコアに基づいて現在のdebounce遅延を予測します。この処理はかなり高度なものです。もちろん、スイッチがオンになっていない場合は、デフォルト値は75msです。

次に、`contexualFilterScore`これは、コンテキストのスコアを表します。copilotは、以前のいくつかのコンテキストが採用されているかどうかを記録し、現在のコンテキストが採用される可能性を単純な線形回帰を使用して予測します。一定のしきい値より小さい場合、ユーザーにヒントを表示しないようにすると、ユーザー体験が向上します。

![image](https://files.mdnice.com/user/13429/c175e519-bb25-4781-a9a3-3ca46628e407.png)

現在のバージョンのしきい値は35％である必要があります。この`contextualFilterEnable`スイッチはデフォルトでオンになっています。

最後に、バックエンドにリクエストを送信する

![image](https://files.mdnice.com/user/13429/18128a56-2953-4871-ac50-fa4367b24c5f.png)

### 1. 概要

この記事では、コードを記述することなく、Amazon API GatewayとAWS Lambdaを使用して、RESTful APIを構築する方法を紹介します。

![image](https://files.mdnice.com/user/13429/7653b58f-20d4-405e-bd88-744a6c622999.png)

## Extract核心ロジック

Extractの最初のレイヤーのロジックは、実際には複雑ではありません。最終的にはpromptオブジェクトを返します。

![image](https://files.mdnice.com/user/13429/6913fceb-3f73-4381-a906-1d6ccde0a896.png)

上の図の赤い枠で囲まれたフィールドは設定から来ていますが、その他のフィールドは`getPrompt`メソッドの返り値から来ています。getPromptはpromptを取得するコアロジックであり、ここではそれを詳しく見ていきます。まずは設定の問題から見てみましょう。

copilot（VSCode）システムでは、MicrosoftのAB実験プラットフォームと統合された多くの設定があります。ファイルでこのようなモジュールを見つけることができます。

```jsx
async fetchExperiments(e, t) {
      const n = e.get(r.Fetcher);
      let o;
      try {
        o = await n.fetch("https://default.exp-tas.com/vscode/ab", {
          method: "GET",
          headers: t
        });
      } catch (t) {
        return i.ExpConfig.createFallbackConfig(e, `Error fetching ExP config: ${t}`);
      }
      if (!o.ok) return i.ExpConfig.createFallbackConfig(e, `ExP responded with ${o.status}`);
      const s = await o.json(),
        a = s.Configs.find(e => "vscode" === e.Id) ?? {
          Id: "vscode",
          Parameters: {}
        },
        c = Object.entries(a.Parameters).map(([e, t]) => e + (t ? "" : "cf"));
      return new i.ExpConfig(a.Parameters, s.AssignmentContext, c.join(";"));
    }
```

これはab実験のプラットフォームを引っ張ってきたもので、多くの特性のスイッチは設定を通して出す、copilotの関連設定も例外ではない。

これらの設定はプラットフォームで指定されない時、デフォルト値である。

私は実際にパケットを取った後、Copilotプラグインの設定が設定プラットフォームを通して指定されていないように見えたので、全体のフィールドはデフォルト値で取られるべきです。

- `suffixPercent`，デフォルト値は15。
- `fimSuffixLengthThreshold`，デフォルト値は0
- `maxPromptCompletionTokens`，デフォルト値は2048
- `neighboringTabsOption`，デフォルト値はeager
- `neighboringSnippetTypes`，デフォルト値はNeighboringSnippets
- `numberOfSnippets`，デフォルト値は4
- `snippetPercent`，デフォルト値は0
- `suffixStartMode`，デフォルト値はCursorTrimStart
- `tokenizerName，` デフォルト値はcushman002
- `indentationMinLength`，デフォルト値はundefined
- `indentationMaxLength`，デフォルト値はundefined
- `cursorContextFix`，デフォルト値はfalse

これらはPromptの基本設定フィールドとしてgetPromptメソッドに渡されます。

## getPromptの核心ロジック

### いくつかの追加設定フィールド

getPromptロジックで、まずいくつかの追加設定フィールドを拡張しています：

- `languageMarker`、デフォルト値はTop
- `pathMarker`、デフォルト値はTop
- `localImportContext`、デフォルト値はDeclarations
- `snippetPosition`、デフォルト値はTopOfText
- `lineEnding`、デフォルト値はConvertToUnix
- `suffixMatchThreshold`、デフォルト値は0
- `suffixMatchCriteria`、デフォルト値は`Levenshtein`
- `cursorSnippetsPickingStrategy`、デフォルト値は`CursorJaccard`

### promptの構成

Copilotでは、promptは複数のタイプで構成されています。PromptElementKindで確認できます。

- `BeforeCursor`、カーソルの前の内容
- `AfterCursor`、カーソルの後の内容
- `SimilarFile`、現在のファイルに似ている内容
- `ImportedFile`：import依存
- `LanguageMarkder`、ファイルの先頭のマークアップ
- `PathMarker`、ファイルのパス情報

### PromptElementの優先順位

Copilotは、優先順位を設定するために、優先順位の補助クラスを実装しました。

```jsx
class Priorities {
  constructor() {
    this.registeredPriorities = [0, 1];
  }
  register(e) {
    if (e > Priorities.TOP || e < Priorities.BOTTOM) throw new Error("Priority must be between 0 and 1");
    this.registeredPriorities.push(e);
    return e;
  }
  justAbove(...e) {
    const t = Math.max(...e);
    const n = Math.min(...this.registeredPriorities.filter(e => e > t));
    return this.register((n + t) / 2);
  }
  justBelow(...e) {
    const t = Math.min(...e);
    const n = Math.max(...this.registeredPriorities.filter(e => e < t));
    return this.register((n + t) / 2);
  }
  between(e, t) {
    if (this.registeredPriorities.some(n => n > e && n < t) || !this.registeredPriorities.includes(e) || !this.registeredPriorities.includes(t)) throw new Error("Priorities must be adjacent in the list of priorities");
    return this.register((e + t) / 2);
  }
}
```

Copilotの中で、異なるタイプの優先度の生成方法は以下のようになっています：

```jsx
const beforeCursorPriority = priorities.justBelow(p.Priorities.TOP);
  const languageMarkerPriority =
    promptOpts.languageMarker === h.Always
      ? priorities.justBelow(p.Priorities.TOP)
      : priorities.justBelow(beforeCursorPriority);
  const pathMarkerPriority =
    promptOpts.pathMarker === f.Always ? priorities.justBelow(p.Priorities.TOP) : priorities.justBelow(beforeCursorPriority);
  const importedFilePriority = priorities.justBelow(beforeCursorPriority);
  const lowSnippetPriority = priorities.justBelow(importedFilePriority);
  const highSnippetPriority = priorities.justAbove(beforeCursorPriority);
```

ここで簡単に推測することができる：

- `beforeCursorPriority`，0.5
- `languageMarkerPriority`，0.25
- `pathMarkderPriority`，0.375
- `importedFilePriority`，0.4375
- `lowSnippetPriority`，0.40625
- `highSnippetPriority`，0.75

だから、デフォルトのシーンで、これらのタイプの優先順位は：`highSnippetPriority` > `beforeCursorPriority` > `importedFilePriority` > `lowSnippetPriority` > `pathMarkderPriority` > `languageMarkerPriority`

### PromptElement主要内容

1. **languageMarker和pathMarker**
    
    languageMarkerとpathMarkerは、promptWishListに最初に配置されたものです。前の分析でわかりましたが、構成では、languageMarkerとpathMarkerにはデフォルト値があります。そのため、次の分岐に入ります。
    
  ```jsx
    if (promptOpts.languageMarker !== h.NoMarker) {
        const e = newLineEnded(r.getLanguageMarker(resourceInfo));
        languageMarkerId = promptWishlist.append(e, p.PromptElementKind.LanguageMarker, languageMarkerPriority);
      }
      if (promptOpts.pathMarker !== f.NoMarker) {
        const e = newLineEnded(r.getPathMarker(resourceInfo));
        if (e.length > 0) {
          pathMarkerId = promptWishlist.append(e, p.PromptElementKind.PathMarker, pathMarkerPriority);
        }
      }
  ```
    
    这两个函数实现也比较简单，我们先来看一下getLanguageMarker：
    
    ```jsx
    exports.getLanguageMarker = function (e) {
      const {
        languageId: t
      } = e;
      return -1 !== n.indexOf(t) || hasLanguageMarker(e) ? "" : t in r ? r[t] : comment(`Language: ${t}`, t);
    };
    ```
    
    这里首先确认了languageId，不在ignoreList当中，在copilot中，有两种语言是被排除在外的：
    
    ```jsx
    const n = ["php", "plaintext"];
    ```
    
    次に言語本体に構文をマークする機能があるかどうかを調べます。このマップには（HTML、Python、Ruby、Shell、YAML）があります。：
    
    ```jsx
    const r = {
      html: "<!DOCTYPE html>",
      python: "#!/usr/bin/env python3",
      ruby: "#!/usr/bin/env ruby",
      shellscript: "#!/bin/sh",
      yaml: "# YAML data"
    };
    ```
    
    それ以外の場合は、このようなコメント行を返します。：
    
    ```jsx
    // Language: ${languageId}
    ```
    
    getPathMarkerロジックはもう少し簡単です。 1行のコメントのみで、ファイルパスを示します（この情報がモデルにどのように役立つかはまだわかりませんが、パスにはディレクトリ構造とファイル名が含まれている可能性があり、モデルが推論をよりよく実行できますか？）：
    
    ```jsx
    exports.getPathMarker = function (e) {
      return e.relativePath ? comment(`Path: ${e.relativePath}`, e.languageId) : "";
    };
    ```
    
2. **localImportContext**
    
    localImportContextの実装はちょっと複雑なので、上記の設定を見ると、デフォルトでも値があることがわかり、下のブランチに入ります。：
    
    ```jsx
    if (promptOpts.localImportContext !== y.NoContext)
        for (const e of await i.extractLocalImportContext(resourceInfo, promptOpts.fs))
          promptWishlist.append(newLineEnded(e), p.PromptElementKind.ImportedFile, importedFilePriority);
    ```
    
    extractLocalImportContextは非同期関数です。実装を見てみましょう。：
    
    ```jsx
    const reg = /^\s*import\s*(type|)\s*\{[^}]*\}\s*from\s*['"]\./gm;
      exports.extractLocalImportContext = async function (resourceInfo, fs) {
        let {
          source: source,
          uri: uri,
          languageId: languageId
        } = resourceInfo;
        return fs && "typescript" === languageId ? async function (source, uri, fs) {
          let language = "typescript";
          let result = [];
          const importEndIndex = function (source) {
            let match;
            let lastIndex = -1;
            reg.lastIndex = -1;
            do {
                match = reg.exec(source);
              if (match) {
                lastIndex = reg.lastIndex + match.length;
              }
            } while (match);
            if (-1 === lastIndex) return -1;
            const nextNewLine = source.indexOf("\n", lastIndex);
            return -1 !== nextNewLine ? nextNewLine : source.length;
          }(source);
          if (-1 === importEndIndex) return result;
          source = source.substring(0, importEndIndex);
          let ast = await i.parseTreeSitter(language, source);
          try {
            for (let node of function (node) {
              let t = [];
              for (let childNode of node.namedChildren) if ("import_statement" === childNode.type) {
                t.push(childNode);
              }
              return t;
            }(ast.rootNode)) {
              let filePath = getTSFilePath(uri, node);
              if (!filePath) continue;
              let namedImports = parseNamedImport(node);
              if (0 === namedImports.length) continue;
              let exports = await getExports(filePath, language, fs);
              for (let e of namedImports) if (exports.has(e.name)) {
                result.push(...exports.get(e.name));
              }
            }
          } finally {
            ast.delete();
          }
          return result;
        }(source, uri, fs) : [];
      }
    ```
    
    まず、この関数はTypescriptの言語を判断して、現在のバージョンのCopilotは、tsファイルのimport依存性だけを処理します。
    この関数が非同期である理由は、ここでimport文の構文木を取得する必要があるため、このプロセスはかなり複雑です。copilotはwasmの方法を使用して、tree-sitterを使用して構文木を解析し、このプロセスは非同期です。
    最後に、copilotはすべてのimportを抽出して、すべてのnamed importに対応するexportコードを返します。つまり、最終的に依存ファイルを検索し、使用されるすべてのexportをコンテキストとして抽出します。
    
3. **snippets**
    
    snippetsの処理は複雑です。Copilotでは、まずsnippetsを取得します。
    
    ```jsx
    const snippets = [
        ...retrievalSnippets,
        ...(promptOpts.neighboringTabs === a.NeighboringTabsOption.None || 0 === neighborDocs.length
          ? []
          : await a.getNeighborSnippets(
              resourceInfo,
              neighborDocs,
              promptOpts.neighboringSnippetTypes,
              promptOpts.neighboringTabs,
              promptOpts.cursorContextFix,
              promptOpts.indentationMinLength,
              promptOpts.indentationMaxLength,
              promptOpts.snippetSelection,
              promptOpts.snippetSelectionK,
              lineCursorHistory,
              promptOpts.cursorSnippetsPickingStrategy
            )),
      ];
    ```
    
    この場合、`retrievalSnippets`は空のままで、`neighboringTabs`は前述の分析で`eager`になっているので、`getNeighborSnippets`を通じてこの配列を取得する。
    
    ここで`neighborDocs`が渡されていることに注意する。これはextractの入口で渡されたもので、対応するコードは：
    
    ```jsx
    let neighborDocs = [];
          let neighborSource = new Map();
          try {
            const t = await u.NeighborSource.getNeighborFiles(ctx, uri, repoUserData);
            neighborDocs = t.docs;
            neighborSource = t.neighborSource;
          } catch (t) {
            telemetry.telemetryException(ctx, t, "prompt.getPromptForSource.exception");
          }
    ```
    
    初期値の場合、ここで取得したfileTypeは`OpenTabs`です。そのため、VSCodeを経由して、現在開いているタブの中に、同じタイプの言語ファイルを含むすべての内容を取得します(アクセス時間順)。対応するコードは以下の通りです。
    
    ```jsx
    exports.OpenTabFiles = class {
      constructor(e, t) {
        this.docManager = e;
        this.neighboringLanguageType = t;
      }
      async truncateDocs(e, t, n, r) {
        const o = [];
        let s = 0;
        for (const a of e) if (!(s + a.getText().length > i.NeighborSource.MAX_NEIGHBOR_AGGREGATE_LENGTH) && ("file" == a.uri.scheme && a.fileName !== t && i.considerNeighborFile(n, a.languageId, this.neighboringLanguageType) && (o.push({
          uri: a.uri.toString(),
          relativePath: await this.docManager.getRelativePath(a),
          languageId: a.languageId,
          source: a.getText()
        }), s += a.getText().length), o.length >= r)) break;
        return o;
      }
      async getNeighborFiles(e, t, n, o) {
        let s = [];
        const a = new Map();
        s = await this.truncateDocs(utils2.sortByAccessTimes(this.docManager.textDocuments), e.fsPath, n, o);
        a.set(i.NeighboringFileType.OpenTabs, s.map(e => e.uri));
        return {
          docs: s,
          neighborSource: a
        };
      }
    };
    ```
    
    次に`getNeighborSnippets`の実装を見てみましょう：
    
    ```jsx
    exports.getNeighborSnippets = async function (
      resourceInfo,
      neighborDocs,
      neighboringSnippetTypes,
      neighboringTabs,
      cursorContextFix,
      indentationMinLength,
      indentationMaxLength,
      snippetSelection,
      snippetSelectionK,
      lineCursorHistory,
      cursorSnippetsPickingStrategy
    ) {
      const options = {
        ...exports.neighborOptionToSelection[neighboringTabs],
      };
      const y = (function (
        resourceInfo,
        neighboringSnippetTypes,
        options,
        cursorContextFix,
        indentationMinLength,
        indentationMaxLength,
        lineCursorHistory,
        cursorSnippetsPickingStrategy = i.CursorSnippetsPickingStrategy
          .CursorJaccard
      ) {
        let d;
        if (neighboringSnippetTypes === s.NeighboringSnippets) {
          d =
            void 0 !== indentationMinLength && void 0 !== indentationMaxLength
              ? o.IndentationBasedJaccardMatcher.FACTORY(
                  indentationMinLength,
                  indentationMaxLength,
                  cursorContextFix
                )
              : o.FixedWindowSizeJaccardMatcher.FACTORY(
                  options.snippetLength,
                  cursorContextFix
                );
        } else {
          if (neighboringSnippetTypes === s.NeighboringFunctions) {
            d = o.FunctionJaccardMatcher.FACTORY(
              options.snippetLength,
              cursorContextFix,
              indentationMinLength,
              indentationMaxLength
            );
          } else {
            r.ok(
              void 0 !== lineCursorHistory,
              "lineCursorHistory should not be undefined"
            );
            d = i.CursorHistoryMatcher.FACTORY(
              options.snippetLength,
              lineCursorHistory,
              cursorSnippetsPickingStrategy,
              cursorContextFix
            );
          }
        }
        return d.to(resourceInfo);
      })(
        resourceInfo,
        neighboringSnippetTypes,
        options,
        cursorContextFix,
        indentationMinLength,
        indentationMaxLength,
        lineCursorHistory,
        cursorSnippetsPickingStrategy
      );
      return 0 === options.numberOfSnippets
        ? []
        : (
            await neighborDocs
              .filter((e) => e.source.length < 1e4 && e.source.length > 0)
              .slice(0, 20)
              .reduce(
                async (e, t) =>
                  (
                    await e
                  ).concat(
                    (
                      await y.findMatches(t, snippetSelection, snippetSelectionK)
                    ).map((e) => ({
                      relativePath: t.relativePath,
                      ...e,
                    }))
                  ),
                Promise.resolve([])
              )
          )
            .filter((e) => e.score && e.snippet && e.score > options.threshold)
            .sort((e, t) => e.score - t.score)
            .slice(-options.numberOfSnippets);
    };
    ```
    
    この実装では、次の重要な情報を得ることができます。
    
    - neighboringSnippetTypesはデフォルトでNeighboringSnippetsに設定されているため、`FixedWindowSizeJaccardMatcher`のロジックに入ります
    - 戻り値はneighborDocsの内容に基づいて、小さすぎるファイルと大きすぎるファイルをフィルタリングし、findMatchersを通して取得した結果です
    - 最後に、スコアが低いものをフィルタリングしますが、thresholdはデフォルトで0なので、ここではすべてのコンテンツが保持されているはずです
    - スコアに基づいてソートし、大きい4つを選択します（numberOfSnippetsはデフォルトで4です）
    
    次に、`FixedWindowSizeJaccardMatcher`のロジックを見てみましょう。
    
    ```jsx
    class FixedWindowSizeJaccardMatcher extends i.WindowedMatcher {
        constructor(resourceInfo, snippetLength, cursorContextFix) {
          super(resourceInfo, cursorContextFix);
          this.windowLength = snippetLength;
          this.cursorContextFix = cursorContextFix;
        }
        id() {
          return "fixed:" + this.windowLength;
        }
        getWindowsDelineations(e) {
          return o.getBasicWindowDelineations(this.windowLength, e);
        }
        trimDocument(e) {
          return e.source.slice(0, e.offset).split("\n").slice(-this.windowLength).join("\n");
        }
        _getCursorContextInfo(e) {
          return r.getCursorContext(e, {
            maxLineCount: this.windowLength,
            cursorContextFix: this.cursorContextFix
          });
        }
        similarityScore(e, t) {
          return computeScore(e, t);
        }
      }
    ```
    
    ここでの`snippetLength` はeagerの場合はデフォルトで60になるので、snippetは60行を超えない。
    
    このクラスは`WindowedMatcher`を継承しているので、findMatchesはWindowedMatcherにある：
    
    ```jsx
    async findMatches(e, t = i.SnippetSelectionOption.BestMatch, n) {
          if (t == i.SnippetSelectionOption.BestMatch) {
            const t = await this.findBestMatch(e);
            return t ? [t] : [];
          }
          return t == i.SnippetSelectionOption.TopK && (await this.findTopKMatches(e, n)) || [];
        }
    ```
    
    ここで、第二引数はデフォルトでundefinedなので、BestMatchの分岐になります：
    
    ```jsx
    async findBestMatch(e) {
          if (0 === e.source.length || 0 === this.referenceTokens.size) return;
          const t = e.source.split("\n");
          const n = this.retrieveAllSnippets(e, s.Descending);
          return 0 !== n.length && 0 !== n[0].score ? {
            snippet: t.slice(n[0].startLine, n[0].endLine).join("\n"),
            semantics: o.SnippetSemantics.Snippet,
            provider: o.SnippetProvider.NeighboringTabs,
            ...n[0]
          } : void 0;
        }
    ```
    
    BestMatchというのは、`retrieveAllSnippets` の第0条の結果をスニペットとして返すということです。
    
    ```jsx
    retrieveAllSnippets(e, t = s.Descending) {
          const n = [];
          if (0 === e.source.length || 0 === this.referenceTokens.size) return n;
          const sourceArr = e.source.split("\n");
          const key = this.id() + ":" + e.source;
          const result = c.get(key) ?? [];
          const noCache = 0 == result.length;
          const tokens = noCache ? sourceArr.map(this.tokenizer.tokenize, this.tokenizer) : [];
          for (const [index, [startLine, endLine]] of this.getWindowsDelineations(sourceArr).entries()) {
            if (noCache) {
              const e = new Set();
              tokens.slice(startLine, endLine).forEach(t => t.forEach(e.add, e));
              result.push(e);
            }
            const r = result[index];
            const s = this.similarityScore(r, this.referenceTokens);
            n.push({
              score: s,
              startLine: startLine,
              endLine: endLine
            });
          }
          if (noCache) {
            c.put(key, result);
          }
          return this.sortScoredSnippets(n, t);
        }
    ```
    
    このコードのコアは、ウィンドウを基準に、異なるコードスニペットと現在のファイルの類似度を計算し、ソートされたスニペットリストを返すことです。

    まず、すでに計算されたコードの類似度をキャッシュするために、このスニペットが行われています。

    次に、いくつかのロジックに焦点を当てます。

    - tokenizeを使用して、現在のコードスニペットの各行のトークンを取得します。
    - getWindowsDelineationsを使用して、コードを異なる小窓に分割します（ステップは1です）。
    - 各ウィンドウのトークンと現在のファイル（referenceDoc）のトークンを使用して、類似度の計算を行います（`Jaccard`類似度）。
    
    これらの3つのポイントはすべて非常に重要です。私たちは、それらを分析するために展開します。
    
    1. **tokenize计算每一行的token**
        
        ```jsx
        const p = new Set(["we", "our", "you", "it", "its", "they", "them", "their", "this", "that", "these", "those", "is", "are", "was", "were", "be", "been", "being", "have", "has", "had", "having", "do", "does", "did", "doing", "can", "don", "t", "s", "will", "would", "should", "what", "which", "who", "when", "where", "why", "how", "a", "an", "the", "and", "or", "not", "no", "but", "because", "as", "until", "again", "further", "then", "once", "here", "there", "all", "any", "both", "each", "few", "more", "most", "other", "some", "such", "above", "below", "to", "during", "before", "after", "of", "at", "by", "about", "between", "into", "through", "from", "up", "down", "in", "out", "on", "off", "over", "under", "only", "own", "same", "so", "than", "too", "very", "just", "now"]);
        const d = new Set(["if", "then", "else", "for", "while", "with", "def", "function", "return", "TODO", "import", "try", "catch", "raise", "finally", "repeat", "switch", "case", "match", "assert", "continue", "break", "const", "class", "enum", "struct", "static", "new", "super", "this", "var", ...p]);
        
        tokenize(e) {
          return new Set(splitIntoWords(e).filter(e => !this.stopsForLanguage.has(e)));
        }
        
        function splitIntoWords(e) {
          return e.split(/[^a-zA-Z0-9]/).filter(e => e.length > 0);
        }
        ```
        
        トークンを処理することは、実際には単語分割のプロセスであることがわかります。ただし、通常の単語分割とは異なり、よく使用されるキーワード（if、forなど）をフィルタリングするというステップが追加されます。これらのキーワードは、類似性の計算に影響しません。
        
    2. **getWindowsDelineations分割窗口**
        
        ```jsx
        exports.getBasicWindowDelineations = function (e, t) {
          const n = [];
          const r = t.length;
          if (0 == r) return [];
          if (r < e) return [[0, r]];
          for (let t = 0; t < r - e + 1; t++) n.push([t, t + e]);
          return n;
        };
        ```
        
        `getWindowsDelineations` は複雑ではなく、渡されたwindowSizeに応じて2次元配列を返します。この2次元配列の各要素は、開始行数と終了行数で構成されています。この関数は、ファイル内のwindowSizeの間隔で、すべての可能な間隔を返します。
        
        これらの間隔を取得した後、現在のコンテンツ（同じwindowSize）と類似度の計算を行い、最も高い類似度の間隔のコンテンツを選択して返します。このコンテンツは、最終的なスニペットです。
        
        ここで、現在のコンテンツを取得する方法は：
        
        ```jsx
        get referenceTokens() {
          if (void 0 === this._referenceTokens) {
            this._referenceTokens = this.tokenizer.tokenize(this._getCursorContextInfo(this.referenceDoc).context);
          }
          return this._referenceTokens;
        }
        
        exports.getCursorContext = function e(doc, opts = {}) {
            const opts = function (e) {
              return {
                ...i,
                ...e
              };
            }(opts);
            const s = r.getTokenizer(opts.tokenizerName);
            
            if (void 0 === opts.maxTokenLength && void 0 !== opts.maxLineCount) {
              const e = doc.source.slice(0, doc.offset).split("\n").slice(-opts.maxLineCount);
              const n = e.join("\n");
              return {
                context: n,
                lineCount: e.length,
                tokenLength: s.tokenLength(n),
                tokenizerName: opts.tokenizerName
              };
            }
        		// ...
          };
        ```
        
        可以看到，这里取的是当前光标前所有内容在窗口大小的截断，这个会token分词之后与对应的相关文件token进行相似度计算。
        
    3. **相似度计算（`Jaccard`）**
        
        Copilot通过一个非常简单的**`Jaccard`** 相似度计算方法：
        
        ```jsx
        function computeScore(e, t) {
            const n = new Set();
            e.forEach(e => {
              if (t.has(e)) {
                n.add(e);
              }
            });
            return n.size / (e.size + t.size - n.size);
          }
        ```
        
        实际上，Jaccard相似度计算公式为：
        
        ![image](https://files.mdnice.com/user/13429/c038f44c-7dce-4a3a-9de8-2dc4dc518b4b.png)  
      
        这是一个非常简单的集合运算，利用交集占比来求相似度，Copilot利用两个分词集合来快速计算文本相似度。
        
    
    最后，copilot调用了processSnippetsForWishlist，将snippet加入到wishList当中：
    
    ```jsx
    function $() {
        const maxSnippetLength = Math.round((promptOpts.snippetPercent / 100) * promptOpts.maxPromptLength);
        c.processSnippetsForWishlist(
          snippets,
          resourceInfo.languageId,
          tokenizer,
          promptOpts.snippetProviderOptions,
          {
            priorities: priorities,
            low: lowSnippetPriority,
            high: highSnippetPriority,
          },
          promptOpts.numberOfSnippets,
          maxSnippetLength
        ).forEach((e) => {
          let t = p.PromptElementKind.SimilarFile;
          if (e.provider === c.SnippetProvider.Retrieval) {
            t = p.PromptElementKind.RetrievalSnippet;
          } else {
            if (e.provider == c.SnippetProvider.SymbolDef) {
              t = p.PromptElementKind.SymbolDefinition;
            }
          }
          promptWishlist.append(e.announcedSnippet, t, e.priority, e.tokens, e.normalizedScore);
        });
      }
    ```
    
    从前面我们可以得知snippetPercent默认为0，所以这里maxSnippetLength也为0.
    
    我们深入看一下processSnippetsForWishList的实现：
    
    ```jsx
    exports.processSnippetsForWishlist = function (snippets, languageId, tokenizer, snippetProviderOptions, priorities, numberOfSnippets, maxSnippetLength) {
        const {
          reserved: reserved,
          candidates: candidates
        } = selectSnippets(snippets, numberOfSnippets, snippetProviderOptions);
        let d = 0;
        let h = [];
        let highPriorities = priorities.high;
        let lowPriorities = priorities.low;
        function g(snippet, r) {
          const o = announceSnippet(snippet, languageId);
          const c = tokenizer.tokenLength(o);
          let l;
          if (r + c <= maxSnippetLength) {
            l = highPriorities;
            highPriorities = priorities.priorities.justBelow(l);
          } else {
            l = lowPriorities;
            lowPriorities = priorities.priorities.justBelow(l);
          }
          h.push({
            announcedSnippet: o,
            provider: snippet.provider,
            providerScore: snippet.providerScore,
            normalizedScore: snippet.normalizedScore,
            priority: l,
            tokens: c,
            relativePath: snippet.relativePath
          });
          return r + c;
        }
        for (const snippet of [...reserved, ...candidates]) {
          if (h.length >= numberOfSnippets) break;
          d = g(snippete, d);
        }
        l(h);
        h.reverse();
        return h;
      };
    ```
    
    可以看到这里maxSnippetLength影响的是Priority，在这里默认情况下就是lowPriority了。
    
    这里的处理其实本质上是对score进行正则化，重排序，然后返回announcedSnippet，这个announceSnippet就是最后被加入到Prompt文本里的内容：
    
    ```jsx
    function announceSnippet(e, t) {
        const n = s[e.semantics];
        let i = (e.relativePath ? `Compare this ${n} from ${e.relativePath}:` : `Compare this ${n}:`) + "\n" + e.snippet;
        if (i.endsWith("\n")) {
          i += "\n";
        }
        return r.commentBlockAsSingles(i, t);
      }
    ```
    
    可以看到这里，相关文件的snippet会包裹在注释里，并在头部加上一行`Compare this …`的文案，提供给模型。
    
4. **beforeCursor**
    
    beforeCursor的代码比较简单：
    
    ```jsx
    promptWishlist.appendLineForLine(source.substring(0, offset), p.PromptElementKind.BeforeCursor, beforeCursorPriority).forEach((e) =>
          V.push(e)
        );
    ```
    
    注意这里用了appendLineForLine，而不是append，让我们看一下appendLineForLine的实现：
    
    ```jsx
    appendLineForLine(text, kind, priority) {
        const lineArr = (lines = this.convertLineEndings(text)).split("\n");
        for (let i = 0; i < lineArr.length - 1; i++) lineArr[i] += "\n";
        const lines = [];
        lineArr.forEach((line) => {
          if ("\n" === line && lines.length > 0 && !lines[lines.length - 1].endsWith("\n\n")) {
            lines[lines.length - 1] += "\n";
          } else {
            lines.push(line);
          }
        });
        const result = [];
        lines.forEach((text, index) => {
          if ("" !== text) {
            result.push(this.append(text, kind, priority));
            if (index > 0) {
              this.content[this.content.length - 2].requires = [
                this.content[this.content.length - 1],
              ];
            }
          }
        });
        return result;
      }
    ```
    
    实际上这段代码的作用就是将光标前的内容按行append，这样在token有限的情况下，能够按行保留最大的上下文。
    

### wishList的fullfill整合处理

在接下来就是一系列依赖关系的处理：

```jsx
if (h.Top === promptOpts.languageMarker && V.length > 0 && void 0 !== languageMarkerId) {
    promptWishlist.require(languageMarkerId, V[0]);
  }
  if (f.Top === promptOpts.pathMarker && V.length > 0 && void 0 !== pathMarkerId) {
    if (languageMarkerId) {
      promptWishlist.require(pathMarkerId, languageMarkerId);
    } else {
      promptWishlist.require(pathMarkerId, V[0]);
    }
  }
  if (void 0 !== languageMarkerId && void 0 !== pathMarkerId) {
    promptWishlist.exclude(pathMarkerId, languageMarkerId);
  }
```

ここで私があまり理解していないのは、pathMarkerとlanguageMarkerはこのロジックの中で相互排他的であることです。私たちが上記の分析で見ることができるように、pathMarkerの優先度はlanguageMarkerの優先度よりも高いため、ここにexcludeを追加すると、languageMarkerは常に現れないことを意味します。

最后，如果是suffixPercent为0的情况下，代码到这里就直接结束了，调用fullfill方法返回最终的结果：

```jsx
if (0 === promptOpts.suffixPercent || q.length <= promptOpts.fimSuffixLengthThreshold)
    return promptWishlist.fulfill(promptOpts.maxPromptLength);
```

さて、上記の分析を通じてsuffixPercentはデフォルトの値は15であり、0ではないため、suffixのロジックに入ります。

しかし、fullfillの処理を見てから、suffixのロジックは分析しません。:

```jsx
fulfill(maxPromptLength) {
    const promptChoices = new PromptChoices();
    const promptBackground = new PromptBackground();
    const elements = this.content.map((e, t) => ({
      element: e,
      index: t,
    }));
    elements.sort((e, t) =>
      e.element.priority === t.element.priority
        ? t.index - e.index
        : t.element.priority - e.element.priority
    );
    const requires = new Set();
    const excludes = new Set();
    let lastElement;
    const results = [];
    let promptLength = maxPromptLength;
    elements.forEach((e) => {
      const element = e.element;
      const index = e.index;
      if (
        promptLength >= 0 &&
        (promptLength > 0 || void 0 === lastElement) &&
        element.requires.every((e) => requires.has(e.id)) &&
        !excludes.has(r.id)
      ) {
        let tokens = element.tokens;
        const nextElement = (function (e, t) {
          let n;
          let r = 1 / 0;
          for (const i of e)
            if (i.index > t && i.index < r) {
              n = i;
              r = i.index;
            }
          return n;
        })(results, index)?.element;
        if (element.text.endsWith("\n\n") && nextElement && !nextElement.text.match(/^\s/)) {
          tokens++;
        }
        if (promptLength >= tokens) {
            promptLength -= tokens;
            requires.add(r.id);
          element.excludes.forEach((e) => excludes.add(e.id));
          promptChoices.markUsed(element);
          promptBackground.markUsed(element);
          results.push(e);
        } else {
          if (void 0 === lastElement) {
            lastElement = e;
          } else {
            promptChoices.markUnused(e.element);
            promptBackground.markUnused(e.element);
          }
        }
      } else {
        promptChoices.markUnused(element);
        promptBackground.markUnused(element);
      }
    });
    results.sort((e, t) => e.index - t.index);
    let prefix = results.reduce((e, t) => e + t.element.text, "");
    let prefixLength = this.tokenizer.tokenLength(prefix);
    for (; prefixLength > maxPromptLength; ) {
      u.sort((e, t) =>
        t.element.priority === e.element.priority
          ? t.index - e.index
          : t.element.priority - e.element.priority
      );
      const e = u.pop();
      if (e) {
        promptChoices.undoMarkUsed(e.element);
        promptChoices.markUnused(e.element);
        promptBackground.undoMarkUsed(e.element);
        promptBackground.markUnused(e.element);
        if (void 0 !== lastElement) {
          promptChoices.markUnused(lastElement.element);
          promptBackground.markUnused(lastElement.element);
        }
        lastElement = void 0;
      }
      u.sort((e, t) => e.index - t.index);
      prefix = u.reduce((e, t) => e + t.element.text, "");
      prefixLength = this.tokenizer.tokenLength(prefix);
    }
    const f = [...u];
    if (void 0 !== lastElement) {
      f.push(lastElement);
      f.sort((e, t) => e.index - t.index);
      const prefix = f.reduce((e, t) => e + t.element.text, "");
      const prefixLength = this.tokenizer.tokenLength(prefix);
      if (prefixLength <= maxPromptLength) {
        promptChoices.markUsed(l.element);
        promptBackground.markUsed(l.element);
        const promptElementRanges = new PromptElementRanges(f);
        return {
          prefix: prefix,
          suffix: "",
          prefixLength: prefixLength,
          suffixLength: 0,
          promptChoices: promptChoices,
          promptBackground: promptBackground,
          promptElementRanges: promptElementRanges,
        };
      }
      promptChoices.markUnused(l.element);
      promptBackground.markUnused(l.element);
    }
    const m = new PromptElementRanges(u);
    return {
      prefix: prefix,
      suffix: "",
      prefixLength: prefixLength,
      suffixLength: 0,
      promptChoices: promptChoices,
      promptBackground: promptBackground,
      promptElementRanges: m,
    };
  }
```

这个fullfill逻辑核心有两点：

- 首先按照Priority排序（Priority相同按index），处理文本内容，这就意味着，在有限的Token下，Priority越高的文本越能被保障。
- 输出的时候，是按照index排序的，也就是说Priority只用作处理文本的优先级，最终组合的prefix文本的顺序是按照插入wishList的先后顺序的。

所以我们根据前面的分析，可以看到文本优先级是这样的：

- `languageMarker`
- `pathMarkder`
- `importedFile`
- `Snippet`
- `beforeCursor`

而处理优先级是这样的（优先保证的内容）：

- `beforeCursor`
- `importedFile`
- `Snippet`
- `pathMarkder`
- `languageMarker`

### Prompt组成的图示

prompt从代码上看比较复杂，我们整体把prefix的组成画一下做个总结：

![image](https://files.mdnice.com/user/13429/390d7bd0-4fa5-457d-bb0e-9c929aa72ae2.png)

## 抓包实验一下

我们找个TS文件来试试：

![image](https://files.mdnice.com/user/13429/fc5484f9-4d01-4d20-a056-68bbb5256902.png)

可以看到，在Copilot发起的请求中，prompt包含了Path Marker和BeforeCursor两个部分，这也是我们在使用过程中绝大多数的面临场景。

如果代码相关性够高，可以看到snippet的部分，比如我们拷贝了一个简单的文件：

![image](https://files.mdnice.com/user/13429/119e8984-ea31-4b10-89ea-b8773b80b0c9.png)

这个时候就会生成对应的snippet：

![image](https://files.mdnice.com/user/13429/f3748c7d-a605-41d1-bcac-9c16f5e334be.png)

## まとめ

Copilotから学ぶべきことはいくつかあります：

- エディタの入力の境界判定、例えば、少なすぎる、多すぎる、キャンセルなど、さまざまなシーンの考慮が含まれています。
- キャッシュの考え方、マルチキャッシュ戦略を使用してバックエンドを保護し、モデルの計算自体が高価なことを保護します。
- プロンプトの設計、コードのコンテキストだけでなく、ファイルの解析やエディタのオープンに関連するコードにも多くのコードがあります。
- 単純なJaccardアルゴリズムを使用して、トークン化されたテキストの類似度を計算し、現在のコンテキストに関連するスニペットを迅速に決定できます。
- 実験機能、Copilotでは、多くのパラメータ、優先度、設定フィールドが実験を介して制御されます。完全な監視およびレポートシステムがあり、これらのパラメーターを調整して、より良い結果を得るのに役立ちます。

Copilotのロジックは想像以上に複雑で、リバースエンジニアリングの難易度も高いです。解析作業に多くの時間を費やしました。本文に関連するツールチェーンとコードはすでにGithubにアップロードされています。必要な方には役立てていただければ幸いです：

[https://github.com/mengjian-github/copilot-analysis](https://github.com/mengjian-github/copilot-analysis)
