# はじめに
vSphere ESXi/ vCener 仮想環境利用前提で記載しています。

VMゲストOSはubuntu Server ver20.10前提で記載しています。

公式Kubernetes環境を検証環境を効率よく作るための手順を記載しています。

※特にVMゲストOSで利用するubuntuについてはセキュリティを度外視した手順になっています。商用環境には向かない、あくまで動作検証用の環境作成を基本としていますので、
セキュリティ事故があったとしても保証は致しかねますので、その上で必要な情報を参考資料としてご利用ください。

# 1.事前準備

とにかくサクッと環境作る、作り直すことができるようにマスター用のVMテンプレートを最初に作成します。

※KubernetesのVMテンプレートでは以下を考慮して事前に対応しておくのが効率がよいです。

 VMゲストOSのテンプレートレシピ（概要）
 
 ・kubaneters環境でのVMではsawpをoffなど事前にしておかないと動かないのでOFF。
 
 ・自前DNSが準備できない場合は、kubanetersで必要となるノード分の名前解決用として
 想定ノード分のホスト名名前解決用として/etc/hostsファイルに事前記載しておく。
 
 ・VMゲストOS自体のIP設定はDHCPではなくスタティックIP設定（推奨）

以下VMテンプレートレシピ詳細です。
# 1-1.OSインストール
スペック考慮事項※1
	
	・vCPU：2Core以上かな
	・Memory：8Gbyteくらいかな
	・HDD：50GB程度※2
	・スタテックでIP設定、DNS、ゲートウェイアドレスも正確に設定。
	・SSH Serverのみは最低限チェックしてインストール、それ以外は基本不要。
	
※1 ubuntuやkubaneters、kubaneters上で稼働させる想定のPod/コンテナのスペックを公式ページで確認してスペック検討ください。
	
※2 Pod/コンテナのPV(Persistent volume)でホスト内のHDDを利用、検証する場合、それを考慮したHDDサイズが必要。
	(後でVMのHDDを追加、拡張することは問題ないですが、後で拡張するのはそれなりに手順は面倒かとは思います。)
	分散ストレージのrancher社のlonghornなど稼働させる場合も考慮が必要です。

DNSが用意できない場合は/etc/hostsにKubernetes検証環境で必要となるノード数をあらかじめ追記。     

	sudo vim /etc/hosts

例) 127.0.0.1 master.test.lan

	192.168.1.1 master master.test.lan
	192.168.1.2 node1 node1.test.lan
	193.168.1.3 node2 node2.test.lan
	194.168.1.4 node3 node3.test.lan
	192.168.1.10 lb lb.test.lan #外部LoadBarancerを作成する場合などその他必要なものを事前に記載

iptablesレガシーバイナリがインストールされていることを確認しiptablesレガシーバージョンに切り替え
kubaneters公式ページより抜粋

	sudo apt-get install -y iptables arptables ebtables
	sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
	sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
	sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
	sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy

FW無効化※ここはセキュリティを考慮していません。
	
	sudo ufw status
	sudo ufw disable
	sudo ufw status


iptablesにてブリッジモード有効化

	cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
	net.bridge.bridge-nf-call-iptables  = 1
	net.ipv4.ip_forward                 = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	EOF

open-vm-toolsはOSインストール時に標準で入っているはず・・・。
/etc/fstabにて最後のswap関連の行を#でコメントアウト & VM再起動

	vim /etc/fstab

稼働後、SSHで対象VMにログインし動作確認してからVM電源OFFしてベースVM作成完了となります。
必要に応じて電源OFF後にVMテンプレート化しておきます。

	sudo poweroff

#--------------------------------------
# 2.kubaneters用のベースVMを作成
# 2-1.CRI(コンテナランタイム)をインストール
以下はkubaneters公式ページのDockerインストール

※公式を参照して以下バージョンにて

	containerd:1.4.4-1
	docker-ce=5:20.10.6~3-0
	docker-ce-cli=5:20.10.6~3-0

1項で作成したベースVMからVMクローン、もしくはVMテンプレートからVMをデプロイし起動し、SSHでログインします。

	sudo apt-get update && apt-get install -y \
	sudo apt-transport-https ca-certificates curl software-properties-common gnupg2
	
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

	sudo add-apt-repository \
	"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
	$(lsb_release -cs) \
	stable"

	sudo apt list -a containerd.io docker-ce docker-ce-cli
	sudo apt-get update && sudo apt-get install -y \
	containerd.io=1.4.4-1 \
	docker-ce=5:20.10.6~3-0~ubuntu-$(lsb_release -cs) \
	docker-ce-cli=5:20.10.6~3-0~ubuntu-$(lsb_release -cs)

	cat <<EOF | sudo tee /etc/docker/daemon.json
	{
  	"exec-opts": ["native.cgroupdriver=systemd"],
  	"log-driver": "json-file",
  	"log-opts": {
	"max-size": "100m"
  	},
  	"storage-driver": "overlay2"
	}
	EOF

	sudo mkdir -p /etc/systemd/system/docker.service.d
	sudo systemctl daemon-reload
	sudo systemctl restart docker
	sudo systemctl enable docker

	systemctl status docker

CRI(コンテナランタイム)インストールは以上
#--------------------------------------

# 2-2.Kubernetesインストール(ver.1.21.0) 以下はkubaneters公式ページより抜粋

	sudo apt-get update && sudo apt-get install -y apt-transport-https curl
	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

	cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
	deb https://apt.kubernetes.io/ kubernetes-xenial main
	EOF

	sudo apt update
	sudo apt list -a  kubelet kubeadm kubectl
	sudo apt install -y kubelet=1.21.0-00 kubeadm=1.21.0-00 kubectl=1.21.0-00

	sudo apt-mark hold kubelet kubeadm kubectl

	sudo poweroff
	
Kubernetesインストールを含めたKubernetes用のベースVM作成は以上となります。
必要に応じてVMテンプレート化します。

# 3.Kubernetes用で必要となるノード分、VMをデプロイしIP・ホスト名・/etc/hostsを記載修正

/etc/hostsファイルの記載修正（ローカルIP部分）し保存
	
	vim /etc/hsots

修正例

	127.0.0.1 node1.test.lan

ホスト名変更
hostnamectl set-hostname <ホスト名>

実行例

	sudo hostnamectl set-hostname node1.test.lan

IPアドレスの設定ファイル修正
/etc/netplan/00-...のファイルをvimで開きIPアドレスを編集し保存

IPアドレス変更を反映※

	sudo netplan apply

※当然ですが反映時点で反映前のIPでのSSH接続ができなくなってしまいます。

IPアドレス変更後のIPでSSH接続してログインできることを確認、その後VM poweroff

	sudo poweroff

Kubernetes用で必要となるnode分のVMが作成出来たら対象分のVMをさらにVMテンプレート化もしくは
VMスナップショットを取得（やり直しがきくように）
これで事前準備が全て完了となります。

# 4.Kubernetesクラスターインストール(master nodeで実行)

	sudo kubeadm init --pod-network-cidr=<Kubernetesクラスターネットワーク> --control-plane-endpoint=<master Node IPアドレス>

実行例（Kubernetesクラスター：ネットワーク10.244.0.0/16 Master NodeのIPアドレス:192.168.1.1の場合）
	
	sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.1.1

Kubernetesクラスタ作成完了後に表示されるnode 追加用のコマンドを控えておく。24時間の有効期限があるため、
有効期限後あるいはコマンドを忘れてしまった場合は以下よりハッシュ値など確認したり作成したりできます。

	kubeadm token list
	sudo kubeadm token create
	openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
	openssl dgst -sha256 -hex | sed 's/^.* //'

以下を実施し、一般ユーザでもkubectlコマンドを利用できるようにします。
	
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	echo 'KUBECONFIG=$HOME/.kube/config' >> $HOME/.bashrc
	source $HOME/.bashrc

以下のコマンドでKubernetesのmaster nodeの稼働状況、CNIがdockerでどうさしているか確認します。

	kubectl get node -A -o wide
	kubectl get pod -A

※CRIがdocker,containerdの場合はkubectl get pod -Aで確認し他時点でcoreDNSはいくら待ってもRunningになりません。
これを解消するために以下CNIのコンテナデプロイが必要になります。


# 5.CNIインストール(master nodeで実行)
※3パターン載せていますのでいずれかを選択しインストールします。

# 5-パターン1 chilinum(ver1.9)インストール(master nodeで実行)
参考URL:https://docs.cilium.io/en/v1.9/gettingstarted/k8s-install-default/

	curl -fsSL -o cilium-quick-install.yaml https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-install.yaml
	kubectl apply -f cilium-quick-install.yaml

公式だと直接デプロイ用のyamlを指定してそのままciliumをインストールだが、後々のことを考えるとなんとなく少し
気持ちが悪いので、インストール用のyamlをローカルにダウンロードしてからインストール。（中身も確認できるので）

ciliumのコンテナがkubectl get pod -Aで全てRunnningになるまで待って完了です。
#ciliumのコンテナが全てRunnningになると、併せてcoreDNSのコンテナも全てRunnningになるかと思います。

	kubectl get pod -A

# 5-パターン2 flannelインストール(master nodeで実行) ※Kubernetes ver1.21.0の場合、APIの宣言が一部古くworningが出ます。

	curl -fsSL -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
	kubectl apply -f kube-flannel.yml

公式だと直接デプロイ用のyamlを指定してそのままciliumをインストールだが、後々のことを考えるとなんとなく少し
気持ちが悪いので、インストール用のyamlをローカルにダウンロードしてからインストール。（中身も確認できるので）

flannelのコンテナが全てRunnningになると、併せてcoreDNSのコンテナも全てRunnningになるかと思います。
	
	kubectl get pod -A

# 6 Kubernetesクラスタにnode追加 (全nodeで実行)
控えておいたnode追加用のコマンドを実行

	sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>

token: kubeadm token listで表示された情報
control-plane-host:master nodeのipアドレス
control-plane-port:デフォルト6443
hash:コマンドを控えていなかった場合、master nodeで以下を実行した結果を記載

	sudo kubeadm token create
	
	openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
	openssl dgst -sha256 -hex | sed 's/^.* //'

実行例
	
	sudo kubeadm join --token rexyhs.g71a1pef4xbwc6t0 192.168.1.1:6443 --discovery-token-ca-cert-hash
	sha256:d5e0b3967d94849f3714a8737dd8d27e591771364afe7c3688ecd813cdb7fa16


# 7 Kubernetesクラスタにnode追加後、nodeのラベルを追加 (master nodeで実行)
kubectl label node <node名> node-role.kubernetes.io/node=

実行例
	
	kubectl label node node1.test.lan node-role.kubernetes.io/node=
	kubectl label node node2.test.lan node-role.kubernetes.io/node=
	kubectl label node node3.test.lan node-role.kubernetes.io/node=

以上でKubernetesクラスタ環境が一通り構築完了となります。
以下は普段よく準備するものかと思いますので、判断の上適宜インストールください。

# オプション1:helmインストール

	curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
	chmod 700 get_helm.sh
	./get_helm.sh

# オプション2:nginx ingressインストール(helm経由でのインストールパターン)
	helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
	helm repo update
	kubectl create ns ingress-system
	helm install nginx-ingress ingress-nginx/ingress-nginx -n ingress-system --set "controller.extraArgs.enable-ssl-passthrough=true"

ingress-systemのネームスペース上にコンテナ稼働させるのため、稼働状況を確認するためにはingress-systemのネームスペースを指定して
稼働を確認するか以下で確認ください。
	
	kubectl get pod -A

その他-その1 cilium用のGUI画面(hubbleインストール)
個人的にはコンテナ間の通信がGUIで見れたりするので便利かと思われます。
※但しcron jobはKubernetes ver 1.21.0系では記載修正が必要そのままではデプロイできない。修正内容も現状煮詰めていません。

	curl -fsSL -o quick-hubble-install.yaml https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-hubble-install.yaml

ダウンロードしたquick-hubble-install.yaml内のUIサービス(hubble-ui)を以下の通りに修正（以下はnodeport 30001で公開）

	vim quick-hubble-install.yaml
	
以下修正箇所は以下サンプル参照
	
	kind: Service
	apiVersion: v1
	metadata:
	  name: hubble-ui
	  labels:
	    k8s-app: hubble-ui
	  namespace: kube-system
	spec:
	  type: NodePort
	  selector:
	    k8s-app: hubble-ui
	  ports:
	    - name: http
	      port: 80
	      targetPort: 8081
	      nodePort: 30001
      
以下にてhubbleコンテナ、サービスをデプロイ

	kubectl apply -f quick-hubble-install.yaml

以下にてhubbleのコンテナとサービスがRunningになったことを確認。
	
	kubectl get pod -A
	kubectl get svc
	
	http://各nodeのIP:30001でGUIにアクセス可能。

# その他-その2 コンテナレジストリ harborインストール(helm経由でのインストールパターン)

	helm repo add harbor https://helm.goharbor.io
	helm repo update
	kubectl create namespace harbor-system

以下実行例です。IPアドレスはnodeのアドレスを指定しています。

	helm install harbor --namespace harbor-system harbor/harbor \
  	--set expose.type=nodePort \
  	--set expose.tls.enabled=false \
  	--set persistence.enabled=false \
  	--set externalURL=http://192.168.1.2:30002 \
  	--set harborAdminPassword=<ログイン時パスワード>

# その他-その3 コンテナ用の分散ストレージlonghornインストール（してみましたのでご参考まで）
参URL:https://longhorn.io/docs/1.1.1/deploy/install/install-with-kubectl/

	curl -fsSL -o longhorn.yaml https://raw.githubusercontent.com/longhorn/longhorn/v1.1.1/deploy/longhorn.yaml

以下longhorn-uiのサービス部分をNodePortに変更

	vim quick-hubble-install.yaml

以下修正例

	aaa
    
以下にてlonghornコンテナ、サービスをデプロイ
	
	kubectl apply -f longhorn.yaml

以下にてhubbleのコンテナとサービスがRunningになったことを確認。

	kubectl get pod -A
	kubectl get svc
	
	http://各ノードIP:31000で管理URLにアクセス可能。

以上です。
