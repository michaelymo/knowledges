# [Consul Architecture](https://www.consul.io/docs/internals/architecture.html)
![10,000 foot view](https://www.consul.io/assets/images/consul-arch-420ce04a.png)

## datacenters
Consul has first class support for multiple datacenters and expects this to be the common case.

## servers
・可用性やパフォーマンスを確保するため、３～５台のサーバで運用するのが望ましい
※追加されるマシンが増えるにつれてコンセンサスは次第に遅くなる
・各データセンター内のサーバは１つのリーダーを選出する
・リーダーはすべてのクエリとトランザクションの処理を担当する
・トランザクションはconsensus protocol一部としてすべてのピアに複製する必要がある
・リーダー以外のサーバがRPC要求を受信すると、リーダー以外のサーバはクラスタリーダーに転送する

### WAN gossip pool
・サーバノードはWAN gossip poolも扱う。
・WAN gossip poolはLAN gossip poolとは異なる
・WAN gossip poolは、データセンターが低接触で互いを発見できるようにすることを目的としている
・サーバーが別のデータセンターあての要求を受信すると、サーバーは正しいデータセンターのランダムサーバーにそのデータを転送する
・その後、ローカルリーダーサーバに転送される
・これによりデータセンタ間の結合は非常に低くなるが、障害検出、接続キャッシング、多重化のため、データセンター間の要求は比較的高速で信頼性がある

### 



## clients
・クライアントの数に制限はなく、数千から数万に用意に拡張できる

## gossip protocol
・データセンタ内のすべてのノードはgossip protocolに参加する
・特定のデータセンタのすべてのノードを含むゴシッププールが存在する
### gossip protocolを使用する目的
・clientは自動でserverのIPアドレスを発見する（設定が不要）
・ノード障害を検出する機能をサーバ上に置かず、分散して実現する（heartbeating schemesよりもスケーラブル）
・通知を行うためのメッセージングレイヤーとして使用する（leader electionなど）

### data
・一般的に、データは異なるデータセンタ間で複製されない。　　
・別のデータセンター内のリソースに対して要求が行われると、ローカルConsulサーバーはそのリソースのリモートConsulサーバーにRPC要求を転送し、結果を返す　　
・リモートデータセンターが利用できない場合、それらのリソースも利用できないが、ローカルデータセンターには影響しない　　
・Consulの組み込みACL複製機能やconsul-replicateのような外部ツールなど、データの一部複製できる特別な場合がある
