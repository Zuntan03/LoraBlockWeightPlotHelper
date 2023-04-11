# LoRA Block Weight Plot Helperチュートリアル

LoRAの階層適用を手助けするツールと、顔LoRAに階層適用するチュートリアルです。

<!-- TODO: 概要は[こちら]()をご覧ください。 -->
![GobuhimeInputGrid](./image/GobuhimeInputGrid.png)
![GobuhimeFix](./image/GobuhimeFix.png)

[Stable Diffusion web UI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)（以降WebUI表記）の[LoRA Block Weight拡張](https://github.com/hako-mikan/sd-webui-lora-block-weight#%E6%A6%82%E8%A6%81)で、LoRAの適用具合を調整するチュートリアルです。<br>
調整を手助けするツール「[LoRA Block Weight Plot Helper](https://docs.google.com/spreadsheets/d/1RPjlQOFt8KtNVhigmZB5LV_p-omnxj5LPPB1boYShvg/copy)」も合わせて紹介します。

おまけで[C3Lier（セリア）](https://twitter.com/kohya_tech/status/1643024429126148096)の「[階層別学習率](https://github.com/kohya-ss/sd-scripts#4-apr-2023-202344-release-060)」や「[重みづけプロンプト](https://twitter.com/kohya_tech/status/1644689752787156992)」も使ってみます。

![TutorialResult](./image/TutorialResult.webp)

[LoRA Block Weight Plot Helper](https://docs.google.com/spreadsheets/d/1RPjlQOFt8KtNVhigmZB5LV_p-omnxj5LPPB1boYShvg/copy): Googleスプレッドシートを自分用に「コピーを作成」して使います。
![LoraBlockWeightPlotHelper](./image/LoraBlockWeightPlotHelper.png)

# 動作環境

[WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)に以下の拡張をインストールしておいてください。

- [LoRA Block Weight (sd-webui-lora-block-weight)](https://github.com/hako-mikan/sd-webui-lora-block-weight#%E6%A6%82%E8%A6%81)
- [a1111-sd-webui-locon](https://github.com/KohakuBlueleaf/a1111-sd-webui-locon.git)
- [SuperMerger (sd-webui-supermerger)](https://github.com/hako-mikan/sd-webui-supermerger.git)
- [Additional Networks (sd-webui-additional-networks)](https://github.com/kohya-ss/sd-webui-additional-networks.git)

# チュートリアルの流れ

1. [Effective Block Analyzerでの下調べ](#effective-block-analyzerでの下調べ)
1. [LoRA Block Weight Plot Helperで各層を詰める](#lora-block-weight-plot-helperで各層を詰める)
1. [さらなる深みへ](#さらなる深みへ)
1. [LoRAへの重みマージ](#loraへの重みマージ)
1. [LoRAへのメタ情報追加](#loraへのメタ情報追加)
1. [その他の機能](#その他の機能)
1. [おまけ](#おまけ)


# Effective Block Analyzerでの下調べ

[LoRA Block Weight](https://github.com/hako-mikan/sd-webui-lora-block-weight#%E6%A6%82%E8%A6%81)はLoRAの各層に対して重み付けをすることで、LoRAの適用具合を調整できます。
各層がどのような描画を担当しているかの大まかな調査結果はありますが、様々な要因の影響を受けるため「このプリセットだけを使っていれば大丈夫」とは言いづらくあります。

このようなLoRAの各層の影響を調査するためにLoRA Block Weightには「[Effective Block Analyzer](https://github.com/hako-mikan/sd-webui-lora-block-weight#effective-block-analyzer-1)」という機能がありますので、まずはこれで各層の影響度合いを調査します。

このチュートリアルでは以下の12枚の画像を[Defmix2](https://huggingface.co/Defpoint/Defmix-v2.0)で学習した「[ゴブ（リン）姫 (./gobuhime-Defmix2.safetensors)](./gobuhime-Defmix2.safetensors)」の顔LoRAで階層を調整してみます。

![GobuhimeInputGrid](./image/GobuhimeInputGrid.png)

緑色の肌は全身タイツのような扱いなのか、服が緑色になったり、服を着ると肌が緑色でなくなったりしますが無視してください。
呼び出しキーワードは「`gobuhime, greenskin, longhair, hairornament, choker`」ですが、悪影響を防ぐために「`greenskin`」を抜いて使っています。

この[ゴブ姫LoRA (./gobuhime-Defmix2.safetensors)](./gobuhime-Defmix2.safetensors)をLoRAモデルのパス(stable-diffusion-webui\models\Lora\など)にコピーしてください。

ではWebUIでEffective Block Analyzerをゴブ姫LoRAに使ってみましょう。

以下のプロンプトを入力して、生成ボタン左下の矢印ボタンを押すと設定が適用されます。
モデル([Defmix2](https://huggingface.co/Defpoint/Defmix-v2.0))やVAE([kl-f8-anime2.ckpt](https://huggingface.co/hakurei/waifu-diffusion-v1-4/tree/main/vae))やTI([EasyNegative](https://huggingface.co/datasets/gsdf/EasyNegative))がない場合は、ダウンロードするか適当に差し替えるかしてください。
```
1girl, gobuhime, longhair, hairornament, choker <lora:gobuhime-Defmix2:1:XYZ>
Negative prompt: EasyNegative, (nude, nipples: 1.3)
Steps: 25, Sampler: DPM++ 2M Karras, CFG scale: 8, Seed: -1, Size: 512x512, Model hash: 31c9be0ee8, Model: Defmix-v2.0, Denoising strength: 0.6, Clip skip: 2, Hires upscale: 1.5, Hires steps: 15, Hires upscaler: Latent (nearest-exact)
```
![LbwEffectiveBlockAnalyzerPrompt](./image/LbwEffectiveBlockAnalyzerPrompt.png)

Effective Block Analyzerでは、LoRA読み込み部分の末尾が「:XYZ>」となっている必要があることに注意してください。

[Effective Block Analyzerが高解像度補助(hires.fix)に対応したのは3/20](https://github.com/hako-mikan/sd-webui-lora-block-weight#updates%E6%9B%B4%E6%96%B0%E6%83%85%E5%A0%B1)ですので、古い場合は更新してください。

次に画面左下にある「LoRA Block Weight」の設定をします。
「Effective Block Analyzer」を選択して「change X-Y」をチェックします。

![LbwEffectiveBlockAnalyzer](./image/LbwEffectiveBlockAnalyzer.png)

この状態で「生成」すると「Effective Block Analyzer」がグリッド画像を生成します。
通常の「X/Y/Zプロット」を有効にしてはいけない、つまり画面左下の「スクリプト」は「なし」にしておく必要がありますので、ご注意ください。

上の行は元画像で、中の行は各層の重みを0.5に減らした画像で、下の行は上と中の差分の画像です。つまり下の画像の色づいている部分が、各層を0.5から1にあげた際に影響が出ている部分になります。

「Effective Block Analyzer」の結果は生成された絵の内容によって傾向が異なりますので、何度か生成して各グリッド画像を見比べます。
例えば以下のような点に気をつけて、自分にとって重要な層、重要でない層、どちらとも言えない層を分類します。

- 全体の変更量（上部の%表記）
- 一番下の差分画像で、どこに変更が集中しているか
	- 顔や髪の毛は顔LoRAにとって重要
	- 背景やポーズの変更は顔LoRAにとって不要

![EffectiveBlockAnalyzer0](./image/EffectiveBlockAnalyzer0.webp)
![EffectiveBlockAnalyzer1](./image/EffectiveBlockAnalyzer1.webp)
![EffectiveBlockAnalyzer2](./image/EffectiveBlockAnalyzer2.webp)

重要な層（重要）、重要でない層（不要）、どちらとも言えない層（不明）のあてをつけられたら、「[LoRA Block Weight Plot Helper](https://docs.google.com/spreadsheets/d/1RPjlQOFt8KtNVhigmZB5LV_p-omnxj5LPPB1boYShvg/copy)」を自分用にコピーして以下のように設定します。

- 「AddSubValue」は「`0.5`（デフォルト値）」
- LoRA Block WeightのTagになる「NAME」はお好みで
- 重要な層は重みを「1」にして「Fix」をチェックする
- 重要でない不要な層は重みを「0」にして「Fix」にチェックする
- どちらとも言えない不明な層は重みを「0.5」にして「Fix」を**チェックしない**

![PlotHelperGobuhime0](./image/PlotHelperGobuhime0.png)

設定が終わったら設定欄の右側の「`Gobuhime:1,0,0.5,1,0.5,0.5,0.5,0,1,1,0,0,0,1,0.5,0.5,1,1,1,1,0.5,0.5,0.5,0.5,0,0.5`」のような文字列を選択してコピーしておきます。

LoRA Block Weightで以下の設定をします。

![LbwEffectiveBlockAnalyzerDisable](./image/LbwEffectiveBlockAnalyzerDisable.png)

- 「Effective Block Analyzer」を「Disable」する
- Weights settingに先程コピーした「`Gobuhime:数字26個`」を先頭にペーストする
	- 「Save Presets」と「Reload Tags」を押すと上の「Active」横に「Gobuhime」が表示されます。
- プロンプトの「XYZ」を「ALL」に書き換える
```
1girl, gobuhime, longhair, hairornament, choker <lora:gobuhime-Defmix2:1:XYZ>
↓
1girl, gobuhime, longhair, hairornament, choker <lora:gobuhime-Defmix2:1:ALL>
```

次に左下の「スクリプト」で「X/Y/Z plot」を選択して、「X軸の種類」を「Prompt S/R」に、「X軸の値」を「`ALL, Gobuhime`」にして「生成」します。

![XyzProt0](./image/XyzProt0.png)

うまく設定できていれば、以下のような絵が生成されます。

髪の毛の色にじみが軽減されたり、服への緑の色移りが軽減されたりしています。

![GobuhimeAll](./image/GobuhimeAll.png)

# LoRA Block Weight Plot Helperで各層を詰める

LoRA Block Weight Plot Helperでは決定していない（「Fix」していない）各層の重みを、「AddSubValue」で足したり引いたりしたX/Y/Z plotを簡単に生成できます。

今回はLoRAの色にじみや色移りといった悪影響を減らすために、どこまで各層の重みを減らせるか試してみます。

重みを減らすので、プロンプトのLoRA Black WeightのTagを「ALL」から「Sub」にします。

```
1girl, gobuhime, longhair, hairornament, choker <lora:gobuhime-Defmix2:1:ALL>
↓
1girl, gobuhime, longhair, hairornament, choker <lora:gobuhime-Defmix2:1:Sub>
```

LoRA Block Weight Plot Helperの「SubWeights」をLoRA Block Weightに、「SubPromptSR」を「X/Y/Z plot」の「Prompt S/R」の値にそれぞれコピペします。
減算と加算や加算のみを扱う場合は、下の「SubAdd」や「Add」を使います。

![PlotHelperOutput](./image/PlotHelperOutput.png)

「SubWeights」をLoRA Block Weightの「Weights setting」にコピペする際には、設定済みのウェイトをどこかにバックアップしておくと、Ctrl+AからCtrl+Vで簡単に貼り付けられますので作業が楽になります。

「Weights setting」を設定したら、「Save Presets」と「Reload Tags」で、上部のタグ一覧が更新できていることを確認します。

![LbwphWeights0](./image/LbwphWeights0.png)

「SubPromptSR」を「X/Y/Z plot」の「Prompt S/R」の値にコピペします。

![LbwphPromptSr0](./image/LbwphPromptSr0.png)

「生成」すると「Fix」していない層の重み「0.5」から「AddSubValue」の「0.5」を引いて「0」にして生成した画像が、層別でグリッドに生成されます。

![Gobuhime0](./image/Gobuhime0.png)

今回は許容できる範囲でLoRAの影響を減らしたいので、重みを減らしても問題がなかった層を「0」にして「Fix」して、同じ手順で画像を生成します。

![PlotHelperGobuhime1](./image/PlotHelperGobuhime1.png)
![Gobuhime0](./image/Gobuhime1.png)

これを繰り返すことで「Fix」した層を増やしていきます。

悩ましくなってきたら、「SubAddWeights」と「SubAddPromptSr」からコピペすることで、各層の減算と加算の両方の結果を確認できます。

![PlotHelperGobuhime2](./image/PlotHelperGobuhime2.png)
![Gobuhime2](./image/Gobuhime2.png)

生成したグリッド画像を見比べて、どんどん「Fix」していきます。

![PlotHelperGobuhime3](./image/PlotHelperGobuhime3.png)
![Gobuhime3](./image/Gobuhime3.png)

すべての層を「Fix」したら、右側にある「WEIGHT」が完成した重みになります。

`Gobuhime:1,0,0,1,0,0,0,0,1,1,0,0,0,1,0,0,1,1,1,1,0.5,0,0,0,0,0`

![PlotHelperGobuhimeFix](./image/PlotHelperGobuhimeFix.png)

![GobuhimeFix](./image/GobuhimeFix.png)

忘れないうちにバックアップしておいたLoRA Block Weightの「Weights setting」を、元に戻しておきましょう。
後の重みマージのために `Gobuhime:1,0,0,1,0,0,0,0,1,1,0,0,0,1,0,0,1,1,1,1,0.5,0,0,0,0,0`のプリセットを付け加えておいてください。

以下はデフォルトのプリセットです。
```
NONE:0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
ALL:1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1
INS:1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0
IND:1,0,0,0,1,1,1,0,0,0,0,0,0,0,0,0,0
INALL:1,1,1,1,1,1,1,0,0,0,0,0,0,0,0,0,0
MIDD:1,0,0,0,1,1,1,1,1,1,1,1,0,0,0,0,0
OUTD:1,0,0,0,0,0,0,0,1,1,1,1,0,0,0,0,0
OUTS:1,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,1
OUTALL:1,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,1
ALL0.5:0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5
```

# さらなる深みへ

LoRAの各層に対して「いる・いらない」で一旦のプリセットは完成しました。
ですが、ここは沼のスタートラインでもあります。

「AddSubValue」を「`0.25`」にして「`1`」にしていた層を減らしたら、もっと良い結果が得られるのではないか？

![PlotHelperGobuhimeFixSub](./image/PlotHelperGobuhimeFixSub.png)
![GobuhimeFixSub](./image/GobuhimeFixSub.png)

逆に「`0`」にしていた層に「`0.25`」を加算したら、もっと良い結果が得られるのではないか？

![PlotHelperGobuhimeFixAdd](./image/PlotHelperGobuhimeFixAdd.png)
![GobuhimeFixSub](./image/GobuhimeFixAdd.png)

じゃあ次は「AddSubValue」を「`0.1`」にしたら？…

また、LoRAは生成するモデルによって結果が大きく異なります。つまり、それぞれのモデルに合った重みがあってもおかしくありません。

![Numa0](./image/Numa0.webp)

さらに、LoRAは学習したモデルと、生成したモデルの組み合わせでも結果が大きく異なります（X軸: 学習モデル、Y軸: 生成モデル）。

右クリックから画像を別タブで開いて拡大してみると、アニメ絵をChillOutMixで学習したChillOutMixの生成画像が特徴的であったり、PVCで学習したLoRAをPVCで生成するとPVCらしさが失われたりしています。

これらの組み合わせに対しても、もちろん層別の重みを調整することができます。

![Numa1](./image/Numa1.webp)

他にもLoRAには、LoRA同士の組み合わせもあります。

層別の重み付けがLoRA同士の組み合わせでちゃんと有効に働いてしまうため、より深い沼に潜る動機が生まれてしまいます。

![Numa2](./image/Numa2.webp)

今回は他のLoRAやControlNetを使用せずに調整したため、そのままの重みでLoRAやControlNetと合わせて使うと押し負けてしまいます。
LoRAの適用強度を単純に強めることで角は復活しましたが、髪の色は押し負けたままですね。

![GobuhimeFixLora](./image/GobuhimeFixLora.png)

層別の重みを調整する際には、実際に利用する環境に合わせて調整しましょう。

# LoRAへの重みマージ

[SuperMerger拡張](https://github.com/hako-mikan/sd-webui-supermerger.git)で調整した重みをモデル自体にマージできます。

モデルにマージするとより簡単に調整後のLoRAを使用できるようになるメリットがあります。しかし、他のLoRAとの組み合わせなどで元の重みを使いたくても戻せないデメリットもあります。ですので、重めの重みでマージしたものを配布するなど、状況に合わせてうまく使ってください。

1. LoRA Block Weightの「Weights setting」にマージしたい重みを記載して、「Save Presets」と「Reload Tags」しておきます。
	- 今回は「`Gobuhime:1,0,0,1,0,0,0,0,1,1,0,0,0,1,0,0,1,1,1,1,0.5,0,0,0,0,0`」をWeights settingに設定しています。
1. WebUI上部のタブから「SuperMerger」-「LoRA」を選びます。
1. 下の「update list」でLoRAリストを更新して、重みをマージしたいLoRAをひとつ選択します。
1. 「LoRAname1:ratio1:Blocks1,...」の欄に「LoRAファイル名:1.0」と表示されるので、重みの「`:Gobuhime`」を付け足します。
1. 「Merge LoRAs」でマージを実行すると、Message欄にマージしたLoRAの保存先が表示されます。
	- [gobuhime-Defmix2_1.0_Gobuhime.safetensor](./gobuhime-Defmix2_1.0_Gobuhime.safetensors)

![SuperMerger](./image/SuperMerger.png)

元のLoRAにLoRA Block WeightでGobuhimeを適用した絵と、Gobuhimeの重みをマージしたLoRAの絵が、大体一致します（xformersなどにより完全には一致していません）。

![SuperMergerResult](./image/SuperMergerResult.webp)

# LoRAへのメタ情報追加

様々な場面でLoRAを利用できるようにするには、利用者側でLoRA Block Weightの重みを指定してもらう必要があります。
重みなどの情報を利用者に伝えるにはREADME.txtなどをLoRAに添えるのが一般的ですが、ファイルが増える分管理が面倒にもなります。

管理するファイルを増やさない選択肢として、[Additional Networks拡張](https://github.com/kohya-ss/sd-webui-additional-networks.git)でLoRAのメタ情報にREADME.txtと同様の情報を追加する方法があります。

1. WebUI上部のタブで「Additional Networks」を選択し、「モデル」からメタ情報を編集するLoRAを選びます。
	- LoRAの数が多い場合は、「モデルのパスのフィルター」で絞り込むこともできます。
1. 下の「編集が有効」にチェックをしてから、メタ情報を入力します。
	- 名前、作成者、キーワード、URL（入力）、カバー画像(512x768)など
1. メタ情報の入力が終わったら「メタデータを保存」します。
1. 保存したLoRAを「Additional Network」で読み込んでメタ情報を確認したり、花札マークのLoRA選択欄の右上にある「Show metadata」でキーワード(ssmd_keywords)を確認したりできます。

![AdditionalNetworks](./image/AdditionalNetworks.png)

![AdditionalNetworksResult](./image/AdditionalNetworksResult.png)

# その他の機能

![ToolLbwEba_SdsLrw](./image/ToolLbwEba_SdsLrw.png)

## LoRA Block Weight - Effective Block Analyzer - Blocks

Fixしていない層を「LoRA Block Weight」-「Effective Block Analyzer」-「Blocks」に設定できるフォーマットで出力します。

## sd-scripts lr_weight

左側で設定している重みを、sd-scriptの[階層別学習率](https://github.com/kohya-ss/sd-scripts/releases/tag/v0.6.0)の引数フォーマットで出力します。

## 重み文字列の読み込み

![Perse](./image/Perse.png)

`Gobuhime:1,0,0.5,1,0.5,0.5,0.5,0,1,1,0,0,0,1,0.5,0.5,1,1,1,1,0.5,0.5,0.5,0.5,0,0.5` や `NIN:1,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,1` といった重みの文字列を、この表で読み込める形に変換します。

「ParseWeight26」でC3Lier、Locon、LoHaの重みを、「ParseWeight17」で標準LoRAの重みを読み込めます。
標準LoRAがサポート指定いない層の重みは「FillValue」で指定します。

## ヒント欄

プルダウンで参考用の重みを切り替えて参照できます。

![Hint](./image/Hint.png)

下部にあるテーブルを編集すると、ヒントの追加や修正や並び替えができます。

![HintTable](./image/HintTable.png)

# おまけ

新機能の[階層別学習率](https://github.com/kohya-ss/sd-scripts/releases/tag/v0.6.0)や[重みづけプロンプトによる学習](https://twitter.com/kohya_tech/status/1644689752787156992)を試してみました。

手違いやバグなどで、誤った情報が含まれている可能性が大いにあります。

## C3Lierの階層別学習率

LoRAの層別適用で散々に沼を彷徨ったので、得られた雰囲気を[C3Lier](https://twitter.com/kohya_tech/status/1643024429126148096)の[階層別学習率](https://github.com/kohya-ss/sd-scripts/releases/tag/v0.6.0)にぶつけてみました。

まずは勘違いしてしまいそうな点についてです。

**LoRA Block Weightの各層の重み指定をそのまま階層別学習率に適用しても、結果は一致しません。**

以下のグリッドの3桁の数値は、それぞれ重要な層と不明な層と不要な層の重みであり、同じ数値は同じ層別重み設定になっています（876なら重要 0.8、不明 0.7、不要 0.6の重み付け）。
同じ数値の左側がLoRA Block Weightでの重み指定で、右側が同じ階層別学習率で学習（他の条件は同じ）したLoRAでの生成結果です。

![LbwLayer](./image/LbwLayer.png)

学習が必要なため試行回数が上げづらい中での、感覚的な感想です。

- 学習率を0にして層を無効にすると、他の層の学習が強まっている気がする。
	- 他の層に学習のしわ寄せがある？0の層を含めると制御しづらい印象。
- 学習ステップ数よりも、学習量に対する反応が素直かもしれない。
	- 学習ステップは「過学習っぽい絵が出るから」とステップを半分にすると意図しない結果になりがち。
		- 学習率を半分にしたほうが、学習の意図は残りやすい印象。
			- ただし、学習を減らしても意図通りに学習が弱まらない、とも考えられる。
	- 元々、学習ステップ数を最小にしつつ、他のLoRAやControlNetと張り合えるように過学習気味の調整にしてはいる。
- 876（重要 0.8,不明 0.7,不要 0.6）、654、432といった、過度に差を付けない学習率はバランスが取れているからか絵が崩れにくい気がする。
	- ただし、層の影響を減らす効果は当然低い。777、555、333にすると悪影響を減らす効果の減退は顕著。
		- 学習量を調整する変数としてはステップ数や学習率よりも使いやすい印象。
- 重要を1、不明を色々、不要を0.1も試してみた(0?1)。
	- 元々、過学習気味な調整で、重要層を1にすると過学習気味に見えるところが変化しない。
- 重要を色々、不明を0.2や0.3、不要を1も試してみた(?21や?31)
	- 不明0.2の感触が良い気がする。
	- 重要を減らしても、他のLoRAと張り合えている。学習率の数値ほどに学習結果は減っていない、のか？

## 彷徨った沼のパン屑

「-Layer」は「Gobuhime」の重みをそのまま層別学習率に適用したもの。LayerにGobuhimeのBlock Weightをさらに掛けてみたり。
![Llr01-Layer](./image/Llr01-Layer.webp)

「L05-」は「Gobuhime」のWeight「0」を「0.5」に底上げした学習率指定。

「C186」は「重要な層を1.0、不明な層を0.8、不要な層を0.6」と以下のように設定。

`"down_lr_weight=0.6,0.8,1,0.8,0.8,0.6,0.6,0.6,1,0.6,0.6,0.6" "mid_lr_weight=1" "up_lr_weight=0.8,0.6,1,1,1,1,0.8,0.8,0.8,0.6,0.6,0.8"`

「1.0」の重みラベル表記が「1」だったり「0」だったりしますが、あしからず。

![Llr02-L05C186](./image/Llr02-L05C186.webp)

「Gobuhimeの学習率をそのまま使って終わり」とならないことを覚悟する。

「Re:ゼロから始める沼生活、もう一回潜れるドン！」です。成れ果てないように広範囲の調査。

服への色移りなどに違いはあるが、なるほど…わからん…

![Llr03-Wide](./image/Llr03-Wide.webp)

このあたりが狙い目かなぁ？数値にあんまり差をつけないほうが、絵が崩れにくい？

![Llr04-Neraime0](./image/Llr04-Neraime0.webp)

衣装LoRAとの組み合わせも試してみる。層別学習率とLoRA Block Weightの両方で。

層別学習率は0.4, 0.3, 0.2といった低い学習率でも衣装LoRAと張り合えている。

![Llr05-Lora0](./image/Llr05-Lora0.webp)

このあたりかなぁ？と思いつつも、Gobuhimeとの構図自由度の差が気になる。

ポーズはほぼ学習させておらず、プロンプトやControlNetでどうとでもなると思いつつも、Gobuhimeの自由度で再現もしっかりできたら、よりいいな、と。

![Llr06-Neraime1](./image/Llr06-Neraime1.webp)

Gobuhimeのような構図自由度を求めて、不要層の学習率を「0」にしてみる。

衣装LoRAと組み合わせると、描画が崩れやすくなってる？

![Llr07-Lora1](./image/Llr07-Lora1.webp)

不用層の学習率「0」は、通常利用パターンでも角が消えたり、耳が増えたりとパッとしない印象。

不明層の学習率が、重要層と離れていると構図に自由が出ている？のかも？

![Llr08-Zero](./image/Llr08-Zero.webp)

重要層を高めの重み、不明層を低めの重み、不用層を「0.1」にしてみる。悪くない印象。

greenskinが復活してきている、がyellow eyeとblushの影響か青ひげみたくなってるのが嫌だなぁ。greenskinは想定外に厄介でした。

このあたりで学習率を減らしても効果が高いので「比例で扱われていたりしないよね？」と不安を覚える。例えば比例だと「842」と「421」が同じ意味になる。

![Llr09-X_32_1](./image/Llr09-X_32_1.webp)

比例なら「777」「555」「333」が同じ結果になる想定だが、そうはならなかった。一安心。

![Llr10-Lora2](./image/Llr10-Lora2.webp)

さて、どのあたりをオススメ設定にするかなぁ…選択肢を増やしすぎても悩むだけだしなぁ…

![Llr11-753](./image/Llr11-753.webp)

このあたりかなぁ、通常のLoRA Block Weightと比べると、学習前に重みを決めなきゃいけないのはやっぱりしんどい。

![Llr12-Last](./image/Llr12-Last.webp)
![Llr13-LastLora](./image/Llr13-LastLora.webp)

- 記念の層別学習率済みLoRA: [gobuhimeC721-Defmix2.safetensors](./gobuhimeC721-Defmix2.safetensors)
	-  `"down_lr_weight=0.1,0.2,0.7,0.2,0.2,0.1,0.1,0.1,0.7,0.1,0.1,0.1" "mid_lr_weight=0.7" "up_lr_weight=0.2,0.1,0.7,0.7,0.7,0.7,0.2,0.2,0.2,0.1,0.1,0.2"`


とかやっていたら「[重みづけプロンプトによる学習に対応しました。](https://twitter.com/kohya_tech/status/1644689752787156992)」が来ました。

これって、アレですよね。<br>
「**さあ顔を上げて…どうか…どうか君たちの旅路に…溢れんばかりの…呪いと祝福を…**」って言われてるやつですよね。

プレフィックスの「_a」は全てのタグ、「_k」は`gobuhime`呼び出しキーワードタグ、「_s」は`greenskin`タグの重みを変えています。
続く数値が「05」なら「`0.5`」、「15」なら「`1.5`」、「30」なら「`3.0`」のタグへの重みづけをしています。
class_tokensは変更していません。

何かが起きていますが、何が起きているのかよくわかりません。

![WeightedTag](./image/WeightedTag.webp)
![WeightedTagLora](./image/WeightedTagLora.webp)

# ライセンス

[MIT License](./LICENSE.txt)です。

This software is released under the MIT License, see [LICENSE.txt](./LICENSE.txt).
