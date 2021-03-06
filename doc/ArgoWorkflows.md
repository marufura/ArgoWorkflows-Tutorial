###### tags: `flt`

# Argo Workflows
## ServiceAccountについて
一部のデータや機能を利用する場合はServiceAccountを設定しておく必要がある。

TODO: やり方がまだよくわかっていない、要調査
- [Service Accounts](https://argoproj.github.io/argo-workflows/service-accounts/)
- [Workflow RBAC](https://argoproj.github.io/argo-workflows/workflow-rbac/)
- [qiita](https://qiita.com/knqyf263/items/ecc799650fe247dce9c5)

# Workflowの書き方
[CloudNative Days Tokyo 2021: Argo Workflows](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=41)

## sample1
![](img/sample1.png)

- Arog Workflows では一つ一つの処理を**template**という概念で管理する
- **spec.entrypoint** は必須でワークフローがどの処理(template)から開始するかを指定する
- ワークフローの流れは**steps**, **dag**で記述する
- 実際の処理の詳細は「**name: whalesay**」に記述
    - 今回はstepsの内部から参照する方法(inlineは後述)

## sample2
![](img/sample2.png)

- sample1からmainのstepsにstep2を追加
    - stepsの中身は基本的に上から書いた順に実行されていく
    - [実行](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=46)すると...?

## sample3
![](img/sample3.png)

- step内の処理をtemplate内に別に書きたくなければinlineでワークフローを書くこともできる。
- 直感的だがstepが多いと可読性が下がるかも
- stepsを記述する際には「- -」と配列をネストさせる必要があることに注意
    - 以下のエラーが出た場合は要注意
        - `Failed to parse workfloe: json: cannot unmarshal object into Go struct field Template.spec.templates.steps of type []map[string]interface{}`


## sample4
![](img/sample4.png)

- 先ほどstepsでネストしていた(mapの配列が必要だった)理由は、stepで並列に動かしたい処理があった場合に対応するため
- 上の例だと2-1,2-2は並列で動く

## sample5
![](img/sample5.png)

- 「- -」が紛らわしくてstepsを書くのが嫌いならdagを使う方法もある
- dependsを使って順番を制御する必要がある
    - dependsを忘れると全てのstepが並列実行される

## sample6
![](img/sample6.png)

- 同じような処理をparameterを変えて処理させたい場合
- inlineで記載しているとparameterの恩恵は受けにくい
- parametersの定義はstep内のtemplateを呼び出すときに定義するのと、呼び出されるtemplate側の両方で定義してあげる必要がある
    - 呼び出す側は「**arguments**」
    - 呼び出される側は「**inputs**」

---
<details>
<summary>Tips</summary>

- 画像の最終行には"}}"が足りていないミスあり
- codeはstep2を追加した版
</details>

---

## sample7
![](img/sample7.png)

- inputs.parametersの定義はデフォルト値を設定できる
    - その場合はstepsのargumentsは省略できる
    - argumentsで定義すれば上書きされる
        - デフォルト値もargumentsも設定されてなければ**ワークフロー登録時にエラー**(inputs.parameters.xxxx was not supplied)

## sample8
![](img/sample8-1.png)
![](img/sample8-2.png)

- 同じような処理をargumentsだけ変えて繰り返し実行した場合に**withItems**でまとめられる
- あくまで同じ階層で複数回実行されるので並列で処理されるということに注意する

## sample9
![](img/sample9.png)

- 処理が成功(or 失敗)した場合にワークフローを自動で削除したい場合
    - **ttlStrategy**を使用する
    - ttlStrategyがないとCompletedなPodで溢れるので設定しておくのがオススメ
    - いちいちWorkflowに書くのが面倒であればコントローラーでTTLのデフォルト値に設定することも可能
        - workflow-controllerが参照するConfigMapで設定
    - ワークフローは内部的にステータスとして**Phase**という状態を持っている。
      - Phaseごとにttlを設定できる
        - secondsAfterCompletion: PhaseがRunningでないもの(Errorなど)
        - secondsAfterSuccess: PhaseがSucceeded
        - secondsAfterFailure: PhaseがFailed

## sample10
![](img/sample10.png)

- ちょっとしたスクリプトを実行したい場合
- containerの代わりにscriptを使うことができる
- shだけじゃなくてpythonなども使える

## sample11
![](img/sample11.png)

- ワークフローの処理中にKubernetesのリソースを作成したい場合
  - resourceを使って定義することが可能
- serviceAccountNameがない場合はnamespace内のdefaultの権限で実行されることになるので、大抵は失敗する
  - serviceAccountNameは適切に設定するように

## sample12
![](img/sample12.png)

- 処理が失敗した際に自動でリトライしたい場合
  - **retryStrategy**を設定
  - リトライ回数は**limit**で指定する
    - 本実行は0番目のリトライとするので実際に実行されるのは最大(limit+1)回
- sample12
  - exit_codeを[0, 1, 1]でランダムに出力して終了
    - 当たりが出るまでリトライを繰り返す

## sample13
![](img/sample13.png)

- retryStrategyの少し高度な設定
  - リトライの感覚や時間の設定を**backoff**を使って細かく設定できる
    - いわゆるexponential backoff
- sample13
  - duration: 待ち時間
  - factor: リトライごとに factor倍(今回だと2倍)されていく
  - maxDuration: maxDurationを超えた時点で終了
    - この例だと1m(60秒)なのでリトライ2回の後、3回目のリトライの前に打ち切られる
      - 10+20 = 30 : SAFE
      - 10+20+40 = 60 : OUT
  - sample13_optional に factor=1.5 を用意
    - 3回目までリトライする

## sample14
![](img/sample14.png)

- 処理が失敗したらリトライせずにフローを進めてほしい場合
  - 失敗を無視したいstepに対して**continueOn**を設定
  - 失敗したフローがあってもワークフロー全体としては成功扱いになる

## sample15
![](img/sample15.png)

- 直前の処理の結果によって処理を分岐させたい場合
  - 条件付き実行は **when** を使うことで実現可能
  - 特定の処理のステータスは **steps.<step名>.status**で取得可能

## sample16
![](img/sample16.png)

- 処理の結果(status)だけではなくて、出力結果による分岐も同じように書ける
- 出力結果は **steps.<step名>.outputs.result** で取得可能
  - resultを取得するには **RBAC的に pods/log** が必要
  - serviceAccountName を設定しないとエラーになることがあるので注意

## sample17

- 各処理ではなくワークフロー全体として成功したか失敗したかをハンドリングして特定の処理をしたい場合
  - **onExit**を設定することで**exit-handler**を設定できる

## [sample18](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=77)

- ワークフローが失敗した時のみonExitで処理したい場合
  - stepsとwhenを使うことで実現可能
    - stepsの内部でないとwhenは使えないことに注意
- sample18は成功の場合のみ掲載

## [sample19](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=78)

- 処理の途中で一旦止めて確認が済んだら続きを実行したい場合
  - stepsの中に**suspend**を入れることによって実現
    - `suspend:{}` とすることで無限sleep状態に
      - 処理を再開するには
        - UI: RESUME をクリック
        - コマンドライン: `argo resume < workflow-name >`

## [sample20](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=80)

- 処理の途中で一定時間だけ間隔を空けたい場合
  - sample19の有限時間バージョン
  - **suspend**で***duration**を設定する
  - durationの時間が過ぎる前にRESUMEで進めることも可能

## [sample21](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=81)

- GithubやS3からアプリケーションのソースを入手して処理させたい場合
  - inputs.artifactsを使うことで簡単にコンテナ内部にソースを持ってくることができる
  - **git**の他にも**s3**,**http**がある
    - httpはGithubのリリースからバイナリを落として配置する場合に便利

## [sample22](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=83)

- sample21 について http を使った例

## [sample23](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=85)

- バックグラウンドでずっとアプリケーションを起動させておいて、各処理からアクセスさせたい場合
  - **daemon**をtrueにすることによってワークフローが処理されている間ずっと動かし続けることができる
- sampleはNginxの例
  - 他にはデータベースを起動させておいて各種テストを走らせるなどの用途にも

## [sample24](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=87)

- 生成したファイルを次の処理に渡す
  - ボリュームを作成して各処理でmountして使い回す方法
  - **volumeClaimTemplates**を利用する

---
<details>
<summary>sample24, 25は生成したファイルを次の処理に渡したい場合の対処方法</summary>

---
方法は色々ある、ここでは3種類紹介
1. resultで渡していく方法
     - sample16 と同様のやり方なので省略
     - ただしバイナリの時に使うことができない
2. ボリュームを作成して各処理で mount して使い回す(sample24)
3. artifact を使う(sample25)
</details>

---

## [sample25](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=89)

- 生成したファイルを次の処理に渡す
  - **artifact**を使う
    - s3互換のオブジェクトストレージを使うことができる
    - 例ではクラスター内のminioを使ってみる

## [sample26](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=94)

- ワークフロー全体で使える変数を定義したい
  - ベタ書きで定義する(オーバーライド可能)
    - argo submit の際に `--parameter message=hoge` とすることで上書きされる

---
<details>
<summary>sample26, 27, 28はワークフロー全体で使える変数を定義したい場合の対処方法</summary>

---
方法は色々ある、ここでは3種類紹介
1. ベタ書きで定義する(オーバーライド可能)
2. configMap から参照して定義する
3. ワークフローのラベルから定義する
</details>

---

## [sample27](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=95)

<!-- TODO -->
- ワークフロー全体で使える変数を定義したい
  - configMap から参照して定義する

## [sample28](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=97)

- ワークフロー全体で使える変数を定義したい
  - ワークフローのラベルから定義する
    - argo submit の際に `--labels message-from-labels=foo` として上書き可能

## [sample29](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=99)

- ワークフローの同時実行数を制御して同時に動かないようにしたい
  - ワークフローが同時に実行されてしまうと不都合な場合
  - 同時実行数を制御しておくとより安全に運用可能
  - 制御するには **synchronization** を使う
    - **semaphore** による制御(sample29)
    - **mutex** により制御(sample30)

## [sample30](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=102)

- ワークフローの同時実行数を制御して同時に動かないようにしたい
  - ワークフローが同時に実行されてしまうと不都合な場合
  - 同時実行数を制御しておくとより安全に運用可能
  - 制御するには **synchronization** を使う
    - **semaphore** による制御(sample29)
    - **mutex** により制御(sample30)

## [sample31](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=104)

- 渡されたパラメーターを加工したい、json から特定の key の value を取り出したい場合
- 組み込みの関数や sprig を使って加工・抽出が可能
  - 組み込みの関数を使う場合は `{{` の部分を `{{=` にする必要がある
    - 全ての箇所で対応しているわけではない

# [ワークフローのテンプレートの管理](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=110)

# [ワークフローの定期実行](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=121)

# [イベントを検知してワークフローを実行させる](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=124)

# [Tips](https://speakerdeck.com/makocchi/how-to-use-argo-workflows?slide=133)