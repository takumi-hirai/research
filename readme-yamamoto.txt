○ndnSIMの入れ方
sudo apt install -y build-essential libsqlite3-dev libcrypto++-dev libboost-all-dev libssl-dev git python-setuptools
sudo apt install -y python-dev python-pygraphviz python-kiwi python-pygoocanvas python-gnome2 python-rsvg ipython
sudo apt install -y pkg-config
mkdir ndnSIM
cd ndnSIM
git clone https://github.com/named-data-ndnSIM/ns-3-dev.git ns-3
git clone https://github.com/named-data-ndnSIM/pybindgen.git pybindgen
git clone --recursive https://github.com/named-data-ndnSIM/ndnSIM.git ns-3/src/ndnSIM

cd ns-3
./waf configure -d optimized
./waf

sudo ./waf install
cd ..

git clone https://github.com/named-data-ndnSIM/scenario-template.git scenario
cd scenario
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

./waf configure

cd ../ns-3
./waf configure --enable-examples
./waf

./waf --run=ndn-simple

参考にしたサイト↓
https://qiita.com/kmdkuk/items/50e04982089e12c9b11b
http://ndnsim.net/2.0/getting-started.html

○実行方法
ndnSIM/ns-3　に移動
./waf --run=myscenario
で実行できる
コンシューマの動きが見たいときは
NS_LOG=ndn.Consumer:ndn.ConsumerVbr ./waf --run=<シナリオ名>
ルーターの動きが見たいときは
NS_LOG=nfd.Forwarder ./waf --run=<シナリオ名>
といった感じ

○編集したソース
ndnSIM/ns-3/src/ndnSIM/examples/<シナリオ名>
	consumerCbrをconsumerVbrに変更
	結果出力ファイルresult.txtの中身を空に
	consumerVbrに追加したattributeから初期のインタレスト要求周期と要求開始時間を設定できる
	Contents Store のMaxSizeを10000から100に変更

ndnSIM/ns-3/src/ndnSIM/examples/topologies/topo-<n>-consumer.txt
	トポロジーを変更するにはこことシナリオを書き換える
	大規模なトポロジーを試すときはシナリオファイルでfor文を使って設定したかったがよくわからなかった

ndnSIM/ns-3/src/ndnSIM/ndn-cxx/src/lp/tags.hpp
	データやインタレストにつけるタグを定義できる
	SpeedTypeTagを追加してデータと要求周期の指示を紐付け

ndnSIM/ns-3/src/ndnSIM/ndn-cxx/src/lp/fields.hpp
	上のタグにフィールドを追加する
	０が等速、１が高速、２（０，１以外）が低速
	わかりにくいのでstring型とかに変えたい

ndnSIM/ns-3/src/ndnSIM/ndn-cxx/src/lp/tlv.hpp
	Tag Length Value (おそらく)
	よくわからなかったがhopcounttagを参考に適当に追加

ndnSIM/ns-3/src/ndnSIM/NFD/deamon/face/generic-link-service.cpp
	これもよくわからなかったけどhopcounttagを参考に適当に追加
	おそらくデータ（パケット？）にタグとフィールドを定義してる

ndnSIM/ns-3/src/ndnSIM/NFD/deamon/fw/forwarder.hpp
	グループクラスを追加してグループ化に用いる情報を保存できるようにした
	データに周期変更の指示をする関数group::judge（シーケンスナンバー）を追加
	結果ファイルへの出力もここで行ってる

ndnSIM/ns-3/src/ndnSIM/NFD/deamon/fw/forwarder.cpp
	上のjudge（）を呼び出している
	何故か１つのデータが多重に呼び出されているので記述箇所は要検討
	多重に呼ばれていてもグループ分けや周期変更に大きな影響はないが結果出力が汚い

ndnSIM/ns-3/src/ndnSIM/apps/consumer-vbr.cpp
	consumerクラスをオーバーライドしたクラス
	元はconsumer-cbr.cppをそのままコピペ
	Constant　Bit Rateでは名前がおかしいので新規ファイルとして作った
	VBRはVariable　Bit　Rateのつもり

ndnSIM/ns-3/src/ndnSIM/apps/consumer-vbr.hpp
	同じく元はconsumer-cbr.hpp
	m_speedフィールドを追加

○引き継ぎ
卒論の方では初期時間差ｇとアクセス間隔が異なっているみたいな風に書いているが
実際はキャッシュアクセスが321->432->543->...という順番でキャッシュアクセス（インタレスト）が送られているため
グループ化の基準シーケンスナンバがずれているためっぽい
インタレストが何で３つずつ送られるのかは不明
（推測ではインタレスト送信時にまだデータが届いていないインタレストを再送しているのかな）
10秒以上シミュレーションできない問題は原因不明

lifetimeを変更するときはforwarder.hppのマクロを変更すればいい
提案手法でない通常時のシミュレーションを行うときはシナリオファイルのConsumerVｂｒをConsumerCbrに書き換えれば良い
今回は時間差でインタレスト要求を始めたが、最初に要求を始めるシーケンスナンバを変更できるみたい
要求周期の変更値５％を変えるときはConsumer-vbr.cppを書き換える
ファイル出力のフォーマットは結構使い勝手が悪いので,をスペースとかに変更するといいかも
ルータのキャッシュ容量をデフォルトの10000から100まで下げている、戻すときはシナリオファイルを変更



