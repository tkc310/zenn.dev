---
title: "microCMS Markdownをパース時にコンポーネントに変換する方法 (≒MDX)"
emoji: "💁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['microCMS', 'JAMstack', 'Next.js', 'TypeScript']
published: true
---


[microCMS Advent Calendar 2021](https://qiita.com/advent-calendar/2021/microcms) 12日目の記事です。

microCMSに投稿したマークダウン文章をNext.jsのSSGビルド時に、タイトル・リンクなどの要素をコンポーネントに変換する方法について紹介します。  

紹介する方法では投稿時の文章にコンポーネントを含む必要がないため `≒MDX` と表現しています。  

```ts
// MDXの場合
<CustomHeading as="h2">title</CustomHeading>
<CustomLink href="https://google.com">link</CustomLink>

// Markdownの場合
# title
[link](https://google.com)
```

MDX文章として記事を管理する方法と比べて、コンポーネントに変更があった場合に記事の方は `# タイトル` のようなマークダウン構文のままで良いため、修正しなくて良いというメリットがあります。  

```ts
e.g. コンポーネント名やpropsのインターフェースを変更
<Heading level={2}>
↓
<CustomHeading as="h2">
```

## 技術構成について

- Next.js (12.0.4)
- TypeScript
- microCMS
- vercel (Node.js Version 14.x)
- next-mdx-remote (3.0.8)

microCMSへの記事投稿をフックにvercelでNext.js SSGビルドが走る仕組みです。  
https://zenn.dev/tkc310/articles/792582ae9ad131

なお、執筆時点でNode.js LTSは16.12.1ですが、vercelは14系までしか対応していないためご注意ください。  

コンポーネントへの変換には [next-mdx-remote](https://github.com/hashicorp/next-mdx-remote) というライブラリを利用しています。  
(hashicorp謹製)  

## 実装方法について

ほぼ、next-mdx-remoteの使い方の解説になってしまいます。

[README example](https://github.com/hashicorp/next-mdx-remote#examples)に記載されている方法を利用します。  

```ts
import { GetStaticProps } from 'next'
import { serialize } from 'next-mdx-remote/serialize'
import { MDXRemote, MDXRemoteSerializeResult } from 'next-mdx-remote'
import ExampleComponent from './example'

const components = { ExampleComponent }

interface Props {
  mdxSource: MDXRemoteSerializeResult
}

export default function ExamplePage({ mdxSource }: Props) {
  return (
    <div>
      <MDXRemote {...mdxSource} components={components} />
    </div>
  )
}

export const getStaticProps: GetStaticProps<MDXRemoteSerializeResult> = async () => {
  const mdxSource = await serialize('some *mdx* content: <ExampleComponent />')
  return { props: { mdxSource } }
}
```

大枠の処理の流れは下記です。  
1. 記事の取得  
`getStaticProps` (SSGビルド時に呼び出される) でmicroCMSからマークダウン文章を取得  
2. mdx文章をhtmlに変換  
`serialize` で `mdxSource` に変換する。  
(mdx -> react elementに変換された文字列 + ライブラリ用のapiが生えたオブジェクト)  
3. 描画用のコンポーネントに渡す    
`<MDXRemote {...mdxSource} components={components}>` のように `mdxSource` を与えるとhydrateしてくれる。  
このとき `components` に、各要素に対応するコンポーネントをマッピングしたオブジェクトを指定する。  
`e.g. { img: (props) => <CustomImage {...props} /> }`

2,3について、詳しく説明していきます。

### mdx文章をhtmlに変換

Next.jsの `getStaticProps` 関数でmicroCMSで投稿したマークダウン文章を取得して、htmlに変換します。  

なお、このライブラリは名前の通りMDXに対応しているためMDX文章でもhtmlに変換してくれます。  

```ts
// getStaticProps
const res = await fetch(url, key);
const article = await res.json();

const mdxSource = await mdx2html(article.body);
```

変換する処理は `mdx2html` のように関数に切り出しています。  

[オプション](https://github.com/hashicorp/next-mdx-remote)としてmdx.jsのAPIが利用できるため、  
下記のプラグインも併用しています。

- `rehypePrism` - コードのシンタックスハイライトを適用  
- `imageSize` - 画像のwidth, height属性を利用できるようにする  

https://mdxjs.com/docs/extending-mdx/

```ts
// mex2html
import { serialize } from 'next-mdx-remote/serialize';
import { MDXRemoteSerializeResult } from 'next-mdx-remote';
import rehypePrism from '@mapbox/rehype-prism';
import imageSize from 'rehype-img-size';

export const mdx2html = async (
  mdxText: string
): Promise<MDXRemoteSerializeResult> => {

  const mdxSource = await serialize(mdxText, {
    mdxOptions: {
      remarkPlugins: [],
      rehypePlugins: [
        [rehypePrism, {}],
        [imageSize, { dir: 'public' }],
      ],
    },
  });
  return mdxSource;
};

export default mdx2html;
```

### 描画用のコンポーネントに渡す

前述の `mdxSource` を `<MDXRemote>` に渡します。  

```ts
interface Props {
  mdxSource: MDXRemoteSerializeResult
}

export default function ExamplePage({ mdxSource }: Props) {
  return (
    <div>
      <MDXRemote {...mdxSource} components={components} />
    </div>
  )
}
```

`components={components}` には、下記のようにhtml変換後の各要素に対応するカスタムコンポーネントをマッピングしたオブジェクトを渡します。  

```ts
const mdxComponents: any = {
  h2: (props) => <CustomHeading as="h2" {...props} />,
  h3: (props) => <CustomHeading as="h3" {...props} />,
  img: ...省略,
  a: ...省略,
};

type Props = {
  as: 'h2' | 'h3' | 'h4' | 'h5' | 'h6';
  children: ReactNode;
};

const CustomHeading: VFC<Props> = ({ as, children }) => {
  const CustomTag = as;
  return <CustomTag>{children}</CustomTag>
}

export default mdxComponents;
```

なお、MDX文章をソースとする場合は下記のようにコンポーネントも定義できます。  

```ts
import { FaBeer } from 'react-icons/fa';

const mdxComponents: any = {
  <FaBeer />,
  h1: ...省略,
};

export default mdxComponents;
```


解説は以上になります。

実装コードは公開していますので、下記からご確認ください。  
https://github.com/tkc310/microCMS_blog

- 記事取得 `getStaticProps`  
https://github.com/tkc310/microCMS_blog/blob/main/src/pages/articles/%5Bid%5D.tsx#L91-L138
- mdx -> html変換 `mdx2html`  
https://github.com/tkc310/microCMS_blog/blob/main/src/utils/mdx2html.ts#L6-L32
- 各要素 x カスタムコンポーネントのマッピング `mdxComponents`  
https://github.com/tkc310/microCMS_blog/blob/main/src/utils/mdxComponents.tsx#L10-L22
- カスタムコンポーネント `CustomHeading`  
https://github.com/tkc310/microCMS_blog/blob/main/src/components/atoms/CustomHeading/index.tsx
- 描画 `MDXRemote`  
https://github.com/tkc310/microCMS_blog/blob/main/src/pages/articles/%5Bid%5D.tsx#L54


## 最後に

今回の記事ではmicroCMSについてあまり触れませんでしたが、和製サービスでUIも分かりやすくオススメです。  
個人ブログに利用していますが、管理画面から登録したAPIスキーマをJSONとして出力・管理できるためチーム開発する際も便利だと思います。  

アップデートも早くキャッチアップしきれていないため、本記事で紹介した内容もスマートに解決できる方法があるかもしれません。

https://microcms.io/
