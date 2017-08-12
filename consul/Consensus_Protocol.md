# [Consensus Protocol](https://www.consul.io/docs/internals/consensus.html)
- ConsulではConsul Server間で情報の一貫性を保つため、Consensus Protocolを使用している
- Consensus ProtocolはRaft Algorithmがベースになっている

## このページに書いてあること
- Consul Serverの状態遷移の説明
- Raftの説明
- どのようにして複数のConsul Server間で情報を共有しているのかの説明
- サーバの動作モード

## 以下で出てくるわかりにくい用語の意味予想
### ログエントリ（log entry）
- KVSに格納する、各サーバのログに関する、キーとバリューの組み合わせだと思われる

### 読み取り（reads）
- HTTP APIで情報を取得する操作（変更はしない）を指していると思われる

## Raft Protocol Overview
- クライアントからのリクエストを、複数のサーバノードで整合性を保ちつつ応答を返すためのアルゴリズム
- このまとめを見るより[Raftのドキュメント（わかりやすい）](http://thesecretlivesofdata.com/raft/)を見た方が早そう
- Raftは[Paxos](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29)をベースにしたアルゴリズム
- [Paxos](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29)に比べて各ノードの持ちうるステータスが少ないため、シンプルでわかりやすいアルゴリズムになっている

### Raftの重要な要素
#### Log
- Raftの主要要素
- ログを各ノード間で複製することで整合性の問題を回避する

#### FSM([Finite State Machine(有限オートマトン)](https://ja.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E3%82%AA%E3%83%BC%E3%83%88%E3%83%9E%E3%83%88%E3%83%B3))
- Raftではサーバノードは「follower」「candidate」「leader」の３つの状態をもちうる

##### followerの動作
- leaderからのログエントリを受け入れる
- 一定時間leaderからログエントリを受けとらなかったら、candidateに昇格する
- leaderを選出するため、candidateに投票をする
- clientからリクエストを受け取ったら、leaderに転送する

##### candidateの動作
- followerに対してleaderになるために投票を要求する
- followerから定足数の票を受け取ったら、leaderに昇格する

##### leaderの動作
- clientからの新しいログエントリを受け取り、そのエントリを耐久性のあるストレージに保存する
- clientからの新しいログエントリを受け取り、そのエントリを全てのfollowerに複製する

#### Peer set
- Consulではすべてのサーバノードがローカルデータセンターのピアーセットに属する
- 各データセンタ毎に3もしくは5のサーバノードでピアーセットを構成することが推奨される（パフォーマンスを維持しつつ可用性をたかめるため）

#### Quorum
- peer set内でサービスを提供するために必要なサーバノードの数を定義している
- 何台までなら落ちても大丈夫というやつ
- 計算式 (peer setを構成しているサーバノードの数)/2+1
- [サーバノードの数とquorumサイズの対応表](https://www.consul.io/docs/internals/consensus.html#deployment-table)

### Consistency Modes
#### default

#### consistent
- 常にleaderがleaderであり続けるモード
- 「leaderはどのノードである」という確認をpeer内のサーバノード同士でやりとりするため、レイテンシが増える

#### stale
- サーバノードがleaderであるかどうかに関わらず、全てのサーバが読み取りに応答できるモード
- 非常に高速でスケーラブルな読み込み
- 返ってくる値が古い可能性がある（leaderと比較すると50ms以内）
- leaderが応答不能になってもcluster内の他のサーバノードが値を返すことができる

