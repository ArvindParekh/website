---
layout: blog
title: "Kubernetes v1.33: 思い描いていたとおりに動作するようになったImage Pull Policy！"
date:  2025-05-12T10:30:00-08:00
slug: kubernetes-v1-33-ensure-secret-pulled-images-alpha
author: >
  [Ben Petersen](https://github.com/benjaminapetersen) (Microsoft),
  [Stanislav Láznička](https://github.com/stlaz) (Microsoft)
translator: >
  [Takuya Kitamura](https://github.com/kfess)
---

## 思い描いていたとおりに動作するようになったImage Pull Policy！

Kubernetesには意外な挙動がいくつか存在しますが、`imagePullPolicy`の挙動もその一つかもしれません。
KubernetesがPodの実行を本質とするものであることを踏まえると、認証が必要なイメージに対してPodのアクセスを制限しようとする際に、10年以上前から[issue 18787](https://github.com/kubernetes/kubernetes/issues/18787)という形で注意点が存在していたことを知ると、意外に思うかもしれません。
この10年越しの問題が解決されるリリースは、非常に興奮すべきものです。

{{< note >}}
このブログ記事全体を通して「Podの認証情報」という用語が頻繁に使われます。
この文脈においては、この用語は、一般的にコンテナイメージのプルを認証するためにPodが利用できる認証情報全体を指します。
{{< /note >}}

## IfNotPresent、たとえ本来持つべきでないとしても

この問題の要点は、`imagePullPolicy: IfNotPresent`という設定が、まさに文字通りの意味でしか動作せず、それ以上のことは一切行ってこなかったという点です。
ここで、とあるシナリオを考えてみましょう。
まず、*Namespace X*内の*Pod A*が*Node 1*にスケジュールされ、プライベートリポジトリから*image Foo*を必要とする状況を考えます。
イメージプル時の認証情報として、このPodは`imagePullSecrets`の*Secret 1*を参照しています。
*Secret 1*には、プライベートリポジトリからイメージをプルするために必要な認証情報が含まれています。
Kubeletは*Pod A*から提供された*Secret 1*の認証情報を使用し、レジストリから*container image Foo*をプルすることになります。
これが意図した(かつ安全な)動作です。

しかし、ここからが興味深いところです。
*Namespace Y*内の*Pod B*も、たまたま*Node 1*にスケジュールされたとします。
このとき、予期しない(かつ潜在的に安全でない)事態が発生します。
*Pod B*は`IfNotPresent`のイメージプルポリシーを指定し、同じプライベートイメージを参照しているかもしれません。
しかし、*Pod B*は`imagePullSecrets`で`Secret 1`(あるいは本例では、いかなるSecretも)を指定していません。
KubeletがこのPodを実行しようとすると、`IfNotPresent`のポリシーが尊重されます。
Kubeletは、*image Foo*がすでにローカルに存在していることを確認し、その*image Foo*を*Pod B*に提供します。
つまり、*Pod B*はそもそも、そのイメージをプルする権限を示す認証情報を一切提供していないにもかかわらず、そのイメージを実行できてしまうのです。

{{< figure
    src="ensure_secret_image_pulls.svg"
    caption="異なるPodによってプルされたプライベートイメージを使用する"
    alt="プライベートイメージへアクセスしようとする2つのPodの処理の図。1つ目のPodはpull secretを持ち、2つ目のPodは持たない。"
>}}

`IfNotPresent`は、イメージがノード上にすでに存在している場合には*image Foo*をプルすべきではありませんが、ノードにスケジュールされたすべてのPodが、過去にプルされたプライベートイメージへアクセスできてしまうというのは、セキュリティ上不適切な構成です。
これらのPodはそもそも、そのイメージをプルする権限を全く与えられていなかったのです。


## IfNotPresent、ただし本来アクセス権がある場合に限る

Kubernetes v1.33では、SIG AuthとSIG Nodeがついにこの(非常に古くからある)問題への対応を開始し、適切な検証が行われるようになりました！
基本的な期待される挙動は変更されていません。
イメージが存在しない場合、Kubeletはそのイメージをプルしようとします。
その際には、各Podが提供する認証情報が使用されます。
この挙動は1.33以前と同様です。

イメージがすでに存在している場合、Kubeletの挙動は変化します。
これからは、KubeletはPodにそのイメージの使用を許可する前に、そのPodの認証情報を検証するようになります。

この機能の改修にあたっては、パフォーマンスとサービスの安定性も考慮されています。
同じ認証情報を使用するPodは、再認証を要求されることはありません。
これは、Podが同じKubernetesのSecretオブジェクトから認証情報を取得している場合には、たとえその認証情報がローテーションされていたとしても、当てはまります。

## Never pull、ただし認証されている場合に限る

`imagePullPolicy: Never`オプションは、イメージを取得しません。
ただし、コンテナイメージがすでにノード上に存在する場合、そのプライベートイメージを使用しようとするすべてのPodは、認証情報の提示が求められ、その認証情報は検証されます。

同じ認証情報を使用するPodは、再認証を要求されることはありません。
一方で、以前にそのイメージのプルに成功した認証情報を提示しないPodには、プライベートイメージの使用が許可されません。

## Always pull、ただし認証されている場合に限る

`imagePullPolicy: Always`は、これまでも意図おりに動作してきました。
イメージが要求されるたびに、そのリクエストはレジストリに送られ、レジストリ側で認証チェックが実行されます。

以前は、プライベートなコンテナイメージが、すでにイメージをプル済みのノード上で他のPodに再利用されないようにする唯一の手段は、Podのアドミッション時に強制的に`Always`のイメージプルポリシーを適用することでした。

幸いにも、この方法はある程度パフォーマンスに優れていました。
プルされるのはイメージそのものではなく、イメージマニフェストだけだったからです。
しかしながら、それでもコストとリスクは存在していました。
新しいロールアウト、スケールアップ、またはPodの再起動の際には、イメージを提供するレジストリが認証チェックのために必ず利用可能でなければならず、その結果、クラスター内で稼働するサービスの安定性において、イメージレジストリがクリティカルパスに置かれることになります。

## 仕組みについて

この機能は、各ノードに存在する永続的なファイルベースのキャッシュに基づいて動作します。
以下は、この機能がどのように動作するかの簡略化された説明です。
完全な仕様については、[KEP-2535](https://kep.k8s.io/2535)をご参照ください。

初めてイメージをリクエストする際の処理の流れは、以下のとおりです:
  1. プライベートレジストリからイメージを要求するPodが、ノードにスケジュールされる。
  1. 要求されたイメージが、当該ノード上に存在しない。
  1. Kubeletは、そのイメージをプルしようとしている状態であることを示す記録を作成する。
  1. Kubeletは、Podにimage pull secretとして指定されたKubernetesのSecretから認証情報を抽出し、それを使用してプライベートレジストリからイメージを取得します。。
  1. イメージのプルに成功すると、Kubeletはその成功を記録する。この記録には、使用された認証情報(ハッシュ形式)および、それらの認証情報を取得するために使われたSecretの情報も含まれる。
  1. Kubeletは、元のプルしようとしている状態であることを示す記録を削除する。
  1. Kubeletは、プルに成功したことを示す記録を後の利用のために保持する。

後に、同じノードにスケジュールされた別のPodが、以前にプルされたプライベートイメージを要求した場合の処理は次のとおりです:
  1. Kubeletは、その新しいPodがプルのために提供した認証情報を確認する。
  1. その認証情報のハッシュ、またはその認証情報の元となったSecretが、以前のプル成功時に記録されたハッシュまたはSecretと一致する場合、そのPodには以前にプルされたイメージの使用が許可される。
  1. 認証情報またはその認証情報の元となるSecretが、そのイメージに関するプル成功記録の中に存在しない場合、Kubeletはその新しい認証情報を使ってリモートレジストリからの再プルを試み、認証フローを開始する。

## 試してみよう

Kubernetes v1.33では、この機能のアルファ版がリリースされました。
実際に試してみるには、バージョン1.33のKubeletにおいて、`KubeletEnsureSecretPulledImages`フィーチャーゲートを有効にしてください。

この機能や追加のオプション設定の詳細については、Kubernetes公式ドキュメントの[イメージの概要ページ](/ja/docs/concepts/containers/images/#ensureimagepullcredentialverification)をご覧ください。

## 今後の予定

今後のリリースにおいて、以下の対応を予定しています:

1. [Kubeletイメージ認証プロバイダ用の投影サービスアカウントトークン](https://kep.k8s.io/4412)との連携を実現します。これにより、ワークロードに特化した新しいイメージプル認証情報の供給元が追加されます。
1. この機能のパフォーマンスを計測し、将来的な変更の影響を評価するためのベンチマークスイートを作成します。
1. 各イメージプル要求のたびにファイルを読み込む必要がなくなるように、インメモリキャッシュ層を実装します。
2. 認証情報の有効期限をサポートし、以前に検証済みの認証情報でも強制的に再認証するようにします。

## 参加するには

これらの変更について詳しく理解するには、[KEP-2535を読む](https://kep.k8s.io/2535)のが最適です。

さらに関わりたい方は、Kubernetes Slackの[#sig-auth-authenticators-dev](https://kubernetes.slack.com/archives/C04UMAUC4UA)チャンネルで私たちにご連絡ください(招待を受けるには[https://slack.k8s.io/](https://slack.k8s.io/)をご確認ください)。
また、隔週水曜日に開催されている[SIG Authのミーティング](https://github.com/kubernetes/community/blob/master/sig-auth/README.md#meetings)への参加も歓迎です。