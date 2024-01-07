---
title: "1日でシンプルなアラーム機能のCLIをGoで作った話"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "CLI", "tool", "cobra"]
published: true
---

## はじめに

こんにちは、[Web エンジニアの tomo](https://twitter.com/tomokn5)です。

皆さんは仕事中、たとえば「あと 30 分で会議が始まる」というような時、どうしていますか？
僕は、仕事に集中しちゃって忘れることが怖いので、よくスマホのアラームをセットしています。

ただ、いちいちスマホのアラームをセットするのは、エンジニアの僕からするとちょっと面倒なんですよね。
ということで、CLI でアラームをセットできたら便利だと思い、Go の勉強がてら作ってみました。

## 作成した「goalarm-cli」の概要説明

GitHub のリポジトリはこちらです。

https://github.com/tomo-kn/goalarm-cli

まず前提として、Go の開発環境が整っていることが必要です。
公式ドキュメントを参考に、Go をインストールしておいてください。

https://go.dev/doc/install

次に、`go install` コマンドで `goalarm-cli` をインストールします。

```bash
$ go install github.com/tomo-kn/goalarm-cli/cmd/goalarm@latest
```

これで、`goalarm` コマンドが使えるようになります。

```bash
$ goalarm
```

### 使い方

アラームをセットするには、`goalarm set [TIME]` コマンドを使います。

```bash
$ goalarm set 23:18
```

これで、23:18 にアラームが鳴ります。

アラームが鳴ったら Enter キーを押すと、アラームが止まります。

アラームが鳴る前に止めたい場合は、`Ctrl + C` で止めてください。

実際に使ってみた様子はこんな感じです。

![](/images/simple-goalarm-cli/image01.gif)

使用時は音が出るので、PC の音量にはくれぐれもご注意ください 🙏

一応、アラームの消し忘れ防止のため、15 分間アラームが鳴り続けると自動で止まる機能をつけています。

### 仕組み

全 115 行のコードで実装できました。

```go
package main

import (
	"bufio"
	"embed"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/gopxl/beep"
	"github.com/gopxl/beep/speaker"
	"github.com/gopxl/beep/wav"
	"github.com/spf13/cobra"
	"golang.org/x/term"
)

//go:embed assets/alarm.wav
var alarmSound embed.FS

func main() {
	var rootCmd = &cobra.Command{Use: "goalarm"}

	var setCmd = &cobra.Command{
		Use:   "set [time]",
		Short: "Set an alarm",
		Args:  cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
				setTime(args[0])
		},
	}


	rootCmd.AddCommand(setCmd)
	rootCmd.Execute()
}

func setTime(timeStr string) {
	layout := "15:04"
	now := time.Now()

	targetTime, err := time.ParseInLocation(layout, timeStr, now.Location())
	if err != nil {
		fmt.Println("Invalid time format. Please use HH:MM.")
		return
	}

	targetTime = time.Date(now.Year(), now.Month(), now.Day(), targetTime.Hour(), targetTime.Minute(), 0, 0, now.Location())

	if targetTime.Before(now) {
		targetTime = targetTime.Add(24 * time.Hour)
		fmt.Printf("Alarm set for %s (tomorrow)\n", targetTime.Format(layout))
	} else {
		fmt.Printf("Alarm set for %s (today)\n", targetTime.Format(layout))
	}

	diff := targetTime.Sub(now)
	fmt.Println("Waiting for alarm... Press Ctrl + C to cancel.")

	time.Sleep(diff)

	done := make(chan bool)
	go playAlarmSound(done)

	fmt.Println("Alarm! The time is now", targetTime.Format(layout))
	fmt.Println("Press 'Enter' to stop the alarm.")

	oldState, err := term.MakeRaw(int(os.Stdin.Fd()))
	if err != nil {
		panic(err)
	}
	defer term.Restore(int(os.Stdin.Fd()), oldState)

	reader := bufio.NewReader(os.Stdin)
	for {
		char, _, err := reader.ReadRune()
		if err != nil {
			panic(err)
		}
		if char == '\r' { // Enter pressed
			done <- true
			break
		}
	}
}

func playAlarmSound(done chan bool) {
	alarmFile, err := alarmSound.Open("assets/alarm.wav")
	if err != nil {
    log.Fatal(err)
	}
	defer alarmFile.Close()

	streamer, format, err := wav.Decode(alarmFile)
	if err != nil {
		log.Fatal(err)
	}
	defer streamer.Close()

	speaker.Init(format.SampleRate, format.SampleRate.N(time.Second/10))

	loop := beep.Loop(-1, streamer)

	go func() {
		select {
		case <-done:
		case <-time.After(15 * time.Minute):
			fmt.Println("15 minutes have passed, stopping the alarm automatically")
      os.Exit(0)
		}
		speaker.Clear() // Clear the speaker to stop playing
	}()

	speaker.Play(loop)
}
```

アラームが鳴ってからの Enter のみ入力を受け付けるために、`term` パッケージを使っています。

そこが若干苦労したポイントですが、基本的には、`setTime` 関数でアラームをセットして、`playAlarmSound` 関数でアラームを鳴らしているだけです。

Go には cobra という CLI フレームワークがあるので、それを使ってコマンドを作っています。

こんなに簡単に CLI が作れるなんて思ってなかったので、割と感動しました。

## まとめ

Go の勉強がてら、1 日でシンプルなアラーム機能の CLI を作ってみました。

「アラームを設定したいけど、スマホのアプリを使うまでではないな...」という時に結構便利です。

僕も最近はこの goalarm を愛用しています笑。

よろしければ、ぜひ使ってみてください！

## 参考文献

公式ドキュメント

- https://pkg.go.dev/golang.org/x/term
- https://pkg.go.dev/embed

GitHub リポジトリ

- https://github.com/spf13/cobra
- https://github.com/gopxl/beep

ブログ記事

- https://zenn.dev/tama8021/articles/22_0627_go_cobra_cli
- https://zenn.dev/awonosuke/articles/47336619a4f039
- https://future-architect.github.io/articles/20210208/
