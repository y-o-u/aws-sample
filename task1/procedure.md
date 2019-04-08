マネージメントコンソールでぽちぽちやっていく時のメモ

## 1. VPCを作成

| CIDR | name |
|----|----|
| 10.0.0.0/16 | vpc-sample |

## 2. internet gatewayを作成

| name | アタッチするVPC |
|----|----|
| sample-igw | vpc-sample |

1で作成したVPCにアタッチする

## 3. subnetを作成

| CIDR | name | AZ |
|----|----|----|
| 10.0.1.0/24 | sample-public-subnet-a | ap-northeast-1a |
| 10.0.2.0/24 | sample-public-subnet-c | ap-northeast-1c |
| 10.0.10.0/24 | sample-private-subnet-a | ap-northeast-1a |
| 10.0.20.0/24 | sample-private-subnet-c | ap-northeast-1c |

## 4. ルートテーブルの反映

| Destination | target | subnet |
|----|----|----|
| 10.0.0.0/16 | local | sample-public-subnet-a <br> sample-public-subnet-c |
| 0.0.0.0/0 | 2で作ったigw | sample-public-subnet-a <br> sample-public-subnet-c |


  * public subnetに対してルートテーブルを紐付け
  * 2で作ったigwをルートテーブルに追加


## 5. NatGatewayの作成

  private subnetにいるインスタンスからインターネット接続させるため

  * EIPをつけてNatGatewayの作成
  * ルートテーブルを作成

| Destination | target | subnet |
|----|----|----|
| 10.0.0.0/16 | local | sample-private-subnet-a <br> sample-private-subnet-c |
| 0.0.0.0/0 | sample-ngw | sample-private-subnet-a <br> sample-private-subnet-c |


## 6. セキュリティグループを作成

| name | 用途 |inbound roule|
|----|----|----|
| sg-web | webサーバーに対するセキュリティを設定する | - sg-gateからのTCP許可. <br> - sg-elbからのhttp接続(80ポート)を許可|
| sg-db | RDSに対するセキュリティを設定する | - sg-webからの3306ポート許可|
| sg-gate | 踏み台サーバーに対するセキュリティを設定する | - ローカルからのssh接続を許可|
| sg-elb | ELBに対するセキュリティを設定する | - publicからのhttp接続を許可|

## 7. EC2を作成

踏み台サーバーとwebサーバー
インスタンスタイプは一番小さいもの

| name | subnet | SG |
|----|----|----|
| sample-gate| sample-public-subnet-a |sg_gate |
| sample-web1| sample-private-subnet-a |sg-web |
| sample-web2| sample-private-subnet-c |sg-web |

## 8. RDSを作成

インスタンスタイプは一番小さいもの
mysql5.6

| name | subnet | MultiAZ|
|----|----|----|
| sample| sample-private-subnet-a <br> sample-private-subnet-c | yes |

* パラメーターグループとオプショングループをデフォルトからコピーし、作成したDBに紐付ける

  デフォルトのものは編集できないため

* サブネットグループ

  private subnetのみにする

* 暗号化,バックアップ,モニタリング,ログのエクスポート,メンテナンス,削除保護の項目はデフォルト状態にする

## 9. ELBを作成

| name | subnet | SG | ポート|
|----|----|----|----|
| sample-elb| sample-public-subnet-a <br> sample-public-subnet-c |sg_elb | 80:80|

* ヘルスチェック

  TCP:80


## 10. 確認等

  * 踏み台サーバー経由でのwebサーバーへのssh接続

    ローカルPCからssh接続を試す

    以下の設定をssh configに記述
    
```
Host sample-web1
  HostName {webサーバーのIP}
  User ec2-user
  IdentityFile ~/Downloads/sample-key-pair.pem
  ProxyCommand ssh -W %h:%p -i ~/Downloads/sample-key-pair.pem -p 22 ec2-user@{踏み台サーバーのPublicIP}

Host sample-web2
  HostName {webサーバーのIP}
  User ec2-user
  IdentityFile ~/Downloads/sample-key-pair.pem
  ProxyCommand ssh -W %h:%p -i ~/Downloads/sample-key-pair.pem -p 22 ec2-user@{踏み台サーバーのPublicIP}
```

  * webサーバーからインターネット接続できるか確認
```
curl checkip.amazonaws.com
```
    作成したNATGatewayのEIPになっていれば良い

  * webサーバーにhttpdとmysqlをインストール,httpd起動
```
yum install httpd mysql
systemctl start httpd
```

  * webサーバーからRDSへの接続確認
```
mysql -u{username} -p -h {rdsのエンドポイント}
```

  * ELB経由でのhttpアクセス確認

    ブラウザで http://{ELB DNS名} にアクセスして TEST PAGEが表示される


  * RDSのフェイルオーバーの確認

    マネージメントコンソールから再起動を実施し、
    アベイラビリティーゾーンが変更されたことを確認。
    また、webサーバーから接続できることも確認。
