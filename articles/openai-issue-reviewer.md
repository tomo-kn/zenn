---
title: "GitHubのイシューをChatGPTでレビューしてくれるGitHub Actionsを作った"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenAI", "ChatGPT", "TypeScript", "GitHubActions"]
published: true
---

## はじめに

こんにちは、[Web エンジニアの tomo](https://twitter.com/tomokn5)です。

皆さんは会社の業務で、たとえば他のエンジニアに対してより的確な指示を出すために、イシュー作成に大幅な時間を取られることはありませんか？
私は、自分が作成したイシューの内容がわかりやすいか、必要な情報が抜け落ちてないかなどを ChatGPT にチェックしてもらうことがあります。

これを自動化できたら便利だな〜と思い、GitHub Actions を自作してみました。

## 作成した「openai-issue-reviewer」君の概要説明

GitHub のリポジトリはこちらです。

https://github.com/tomo-kn/openai-issue-reviewer

まず、`OPENAI_API_KEY` を[公式サイト](https://platform.openai.com/api-keys)で発行してもらい、それを GitHub のリポジトリの Secrets に登録していただきます。

次に、`.github/workflows` 配下に、たとえば `openai-issue-reviewer.yml` みたいなファイルを作ってもらって、以下のように記述します。

```yaml
name: openai-issue-reviewer

on:
  issues:
    types: [edited, labeled]

jobs:
  openai-issue-reviewer:
    runs-on: ubuntu-latest
    steps:
      - uses: tomo-kn/openai-issue-reviewer@v1.0.0 # Please use the latest version.
        name: openai-issue-reviewer
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          ISSUE_LANGUAGE: "ja-JP" # Set to your preferred language. Default is "en" (English).
```

すると、イシュー概要を編集 or イシューにラベルを追加したときにこのアクションが走り、初回のみ

- issue review 3.5
- issue review 4

という 2 つのラベルを作成してくれます。

これらのラベルをつけてこのアクションを走らせることで、ChatGPT がイシューの内容をレビューしてくれます。

### 実際に使ってみた感じ

試しに、ありそうなイシューを ChatGPT に作ってもらって、実験してみました。

https://github.com/tomo-kn/openai-issue-reviewer/issues/11

GPT-3.5 モデルと GPT-4 モデルの、それぞれのレビュー結果は以下の通りです。

| GPT-3.5 モデル                                 | GPT-4 モデル                                   |
| ---------------------------------------------- | ---------------------------------------------- |
| ![](/images/openai-issue-reviewer/image01.png) | ![](/images/openai-issue-reviewer/image02.png) |

やはり GPT-4 モデルの方が、プロンプトに対する回答がより適切になっているように感じます。

プロンプトのコード一部抜粋

```ts
function createPrompt(
  label: string,
  title: string,
  body: string | null | undefined,
  language: string
) {
  let prompt = `IMPORTANT: Entire response must be in the language with ISO code: ${language}.\n\nYou are a seasoned engineer and a professional who reviews issue requirements for clarity and distinctness. Please output in the following format. `;

  // Step 1: Display the classified label
  prompt += `1. Classified Label: ${label}`;

  // Step 2: Review the issue based on the label
  prompt += `2. Issue Review: <Review the issue based on the title and body provided by the user input. Please refer to the following for perspectives on reviewing an issue. `;

  // Add specific review points based on the label
  switch (label.toLowerCase()) {
    case "bug":
      prompt += `
      - Analyze if the cause and reproduction steps of the bug are clearly and accurately described.
      - Assess if the expected outcomes and impact of the bug are well-defined and realistic.
      - Suggest any additional notes or considerations that might enhance the clarity or completeness of the issue.`;
      break;
    case "feature":
      prompt += `
      - Evaluate if the purpose and specifications of the feature are clear, understandable, and well-articulated.
      - Determine if the criteria for completion (Close conditions) are clear, appropriate, and achievable.
      - Recommend any additional notes or considerations that could improve the feature proposal.`;
      break;
    // 中略
    default:
      prompt += `
      - Review if the issue title and description are clear, concise, and effectively communicate the core issue.
      - Assess if the goals and expectations of the issue are well-defined and realistic.
      - Suggest any additional notes or considerations that could enhance the overall clarity and effectiveness of the issue.`;
  }

  prompt += `>`;

  // Step 3: Suggest improvements for a better issue
  prompt += `3. Suggestions for a Better Issue:<Please suggest a better issue based on the above.>`;

  prompt += `## Constraints\n
- Always follow the 3-step format.
- In the 2nd step, the phase of reviewing the issue, the content should be human-readable and understandable.
- In the 3rd step, the phase to propose a better way to write the issue, the issue should be written in a way that is easier to understand than the current issue.`;

  return prompt;
}
```

GPT-3.5 モデルよりもちろんコストはかかりますが、より高い精度のレビューを求める方は、GPT-4 モデルを使うことをおすすめします。

## まとめ

初めての GitHub Actions 作成で、少し手間取ってしまいましたが、TypeScript で書けるのは非常に便利でしたし楽しかったです。

また、ChatGPT の精度もかなり高く、今後もさらに精度が上がっていくことを期待しています。

改善点等ございましたら、ぜひ OSS 活動の一環として、イシューを作成していただいたり、プルリクエストを送っていただけると嬉しいです 🙏

## 参考文献

公式ドキュメント

- https://platform.openai.com/docs/models
- https://docs.github.com/en/rest/issues
- https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-openai-api
- https://docs.github.com/en/actions/creating-actions/publishing-actions-in-github-marketplace

ブログ記事

- ⭐️ https://qiita.com/eno49conan/items/8916db871050073794ff ：@vercel/ncc で GitHub Actions 用のパッケージをビルドする方法
- https://qiita.com/bugfire/items/a2fa85fa58dd20322e3f
