---
title: "おうちKubernetesに音楽ストリーミングサーバー(兼ファイルサーバー)を構築する"
date: 2021-12-02T07:00:00+09:00
draft: false
categories:
- kubernetes
- 自宅サーバー
---

神(Google)は「**Play Music**」と言われた。するとGoogle Play Musicがあった。

神はそのUXを見て、良しとされた。

神はまた言われた。「**YouTube Musicに移行してください**」

UIは使いづらく、バックグラウンド再生できず、ロードは遅くなり、楽曲メタデータは編集できなくなった。

神はお休みになった。

# 概要

**所有している音楽データをアップロードし、インターネット経由で聴く**というサービスでしっくりくるものがないため、自宅Kubernetesクラスタに自前で構築してみます。

- 家庭内LANからファイルサーバーとして使える
- ファイルサーバーにアップロードした音楽データをインターネット経由で聴ける
- ファイルサイズが大きい楽曲はサーバーサイドでリアルタイムに圧縮して配信する

という要件から、以下のような構成にしてみます。

![architecture.png](/images/airsonic/architecture.png)


- 音楽配信サーバーには [Airsonic](https://airsonic.github.io/)を使います
  - Ingress(L7ロードバランサー)経由でインターネットに接続します
  - IngressをTLS終端にします
- ファイルサーバーとしてSambaを構築します
  - Airsonicとストレージを共有します
  - LoadBalancer Service(L4ロードバランサー)経由で家庭内LANに接続し、インターネットからは遮断します

# 構築

## 1. Storage
![storage.png](/images/airsonic/storage.png)

まず初めに、Podからホストマシンのストレージを使うためのPersistentVolume(PV)とPersistentVolumeClaim(PVC)を作成します。
今回は `node1` の `/mnt/hdd` に音楽データとメタデータ(設定、アカウント情報など)を永続化するとします。

<details><summary><font color="blue">pv.yaml▼</font></summary><div>

```yaml:pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: music
spec:
  capacity:
    storage: 1000Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt/hdd/music
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: config
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt/hdd/config
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: music
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1000Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: config
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

</div></details>

サンプルコードでは [local volume](https://kubernetes.io/docs/concepts/storage/volumes/#local) にしているため、この後にデプロイするAirsonicとsambaはnodeAffinityによって `node1` にスケジューリングされる事になります。
物理ストレージが接続されているノードとは別のノードでPodを稼働させたい場合は[nfs volume](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)にすればOKです。
(OS側のNFSマウント設定については割愛します)

```yaml
...
kind: PersistentVolume
...
  accessModes:
    - ReadWriteMany
  nfs:
    path: /music
    server: 192.168.0.XXX
...
```

## 2. Samba
![samba.png](/images/airsonic/samba.png)

次のステップではsambaをデプロイし、k8sクラスタをファイルサーバーとして使えるようにします。

### Deployment


sambaで一番人気のDocker imageである[dperson/samba](https://hub.docker.com/r/dperson/samba) をデプロイします。
上で作成したPVCを `/music` にマウントします。

<details><summary><font color="blue">samba-deployment.yaml▼</font></summary><div>

```yaml:samba-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: samba
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: samba
  template:
    metadata:
      labels:
        app: samba
    spec:
      containers:
        - name: samba
          image: dperson/samba
          args: [
            "-u", "myuser;mypassword", # UPDATE HERE
            "-s", "music;/music;yes;no;no;myuser",
            "-w", "WORKGROUP"
          ]
          ports:
            - containerPort: 139
              protocol: TCP
            - containerPort: 445
              protocol: TCP
          resources:
            limits:
              cpu: "250m"
              memory: "500Mi"
          livenessProbe:
            tcpSocket:
              port: 445
          volumeMounts:
          - name: music
            mountPath: /music
      volumes:
        - name: music
          persistentVolumeClaim:
            claimName: music
```
</div></details>
<br>

### MetalLB
PodをLAN内に公開するため、L4ロードバランサーに相当するLoadBalancer Serviceもデプロイします。
事前準備としてオンプレミス環境用のLoadBalancer実装であるMetalLBを導入します。

　microk8sの場合は以下のコマンド一発で利用可能になります。

```sh
$ sudo microk8s.enable metallb:192.168.0.AAA-192.168.0.BBB
```

> `:` の後はロードバランサーが用いるローカルIPアドレスプールを入力します。


その他の環境では[公式ドキュメント](https://metallb.universe.tf/installation/)を参照してください。

### Service

samba Podの139番と445番ポートをローカルIPアドレス192.168.0.YYYとしてクラスタ外に公開します。
(YYYはMetalLBのアドレスプールの範囲で未使用のアドレスを設定します)

<details><summary><font color="blue">samba-svc.yaml▼</font></summary><div>

```yaml:samba-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: samba
  namespace: default
spec:
  selector:
    app: samba
  type: LoadBalancer
  loadBalancerIP: 192.168.0.YYY # UPDATE HERE
  ports:
    - name: netbios
      port: 139
      targetPort: 139
    - name: smb
      port: 445
      targetPort: 445
```
</div></details>

EXTERNAL-IPが割り当てられている事を確認します。

```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                         AGE
samba        LoadBalancer   10.aaa.bbb.ccc   192.168.0.YYY   139:3xxxx/TCP,445:3xxxx/TCP     1m
```

---

Windows PCであればエクスプローラの `ネットワークドライブの割り当て` から、`¥¥192.168.0.YYY¥music` をネットワークドライブとしてマウントできるようになります。
手持ちの音楽データを `{アーティスト名}/{アルバム名}/{曲名}.{ファイル形式}` のディレクトリ構成でアップロードしておきます。

## 3. Airsonic

![airsonic.png](/images/airsonic/storage.png)

本題である音楽ストリーミングサーバーのAirsonicをデプロイします。

### Deployment

[linuxserver/airsonic](https://hub.docker.com/r/linuxserver/airsonic) のDockerhubドキュメントに沿って実装していきます。
(公式の[airsonic/airsonic](https://hub.docker.com/r/airsonic/airsonic) イメージはメンテナンスされていないようでした)

sanbaでも用いた music PVCを `/music` にマウントします。
またメタデータはコンテナ内の `/config` に保存されるため、config PVCを `/config` にマウントしてこれらのデータが永続化されるようにしておきます。　　

> Airsonicは内蔵DB(HSQLDB)だけでなくPostgresやMySQLなどの外部DBにも対応しています。
外部DBをStatefulSetとしてk8s上に構築すればAirsonicをstatelessにできて良さそうですが、[パフォーマンスに問題を抱えている](https://github.com/airsonic/airsonic/issues/1340)ため今回は見送りました。

<details><summary><font color="blue">airsonic-deployment.yaml▼</font></summary><div>

```yaml:airsonic-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airsonic
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: airsonic
  template:
    metadata:
      labels:
        app: airsonic
    spec:
      containers:
        - name: airsonic
          image: lscr.io/linuxserver/airsonic
          env:
          - name: TZ
            value: Asia/Tokyo
          - name: PUID
            value: "1000"
          - name: PGID
            value: "1000"
          ports:
            - containerPort: 4040
              protocol: TCP
          resources:
            limits:
              cpu: "2"
              memory: "2Gi"
          livenessProbe:
            httpGet:
              path: /login
              port: 4040
            initialDelaySeconds: 60 # CrashLoopBackOffになる場合はこの値を大きくする
          volumeMounts:
          - name: config
            mountPath: /config
          - name: music
            mountPath: /music
      volumes:
        - name: music
          persistentVolumeClaim:
            claimName: music
        - name: config
          persistentVolumeClaim:
            claimName: config
```

</div></details>

### Service

Sambaとは異なりServiceをクラスタ外に公開する必要は無いため、[Headless Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/#headless-service)として実装します。

<details><summary><font color="blue">airsonic-svc.yaml▼</font></summary><div>

```yaml:airsonic-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: airsonic
  namespace: default
spec:
  selector:
    app: airsonic
  type: ClusterIP
  clusterIP: None
  ports:
    - name: http
      port: 4040
      protocol: TCP
```
</div></details>

### Nginx Ingress Controller

AirsonicはWebアプリケーションのため、L7ロードバランサーに相当するIngressを経由してServiceをクラスタ外に公開します。
事前準備としてオンプレミス環境用のIngress実装であるNginx Ingress Controllerを導入します。
導入手順は[公式ドキュメント](https://kubernetes.github.io/ingress-nginx/deploy/)を参照してください。
(microk8sであれば `sudo microk8s.enable ingress` で終わりです)

### Ingress

クラスターエンドポイントIPへのhttpアクセスをairsonicのServiceに転送する設定でIngressをデプロイします。
https化は次のステップで実施します。

<details><summary><font color="blue">airsonic-ingress-http.yaml▼</font></summary><div>

```yaml:airsonic-ingress-http.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airsonic-http
  namespace: default
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: airsonic
            port: 
              number: 4040
```

</div></details>


```sh
$ kubectl get ingress
NAME            CLASS    HOSTS           ADDRESS     PORTS     AGE
airsonic-http   public   *               127.0.0.1   80        1m
```

---

LAN内PCのブラウザから `http://{クラスターエンドポイントIP}` を開くとログイン画面が表示されます。
初期ID/パスワードはadmin/adminなのでログイン後すぐに変更しましょう。
ログインするとsamba経由でアップロードした音楽を聴くことができるようになっています。

<img src="/images/airsonic/webui.png" width="480">

## 4. https化

![https.png](/images/airsonic/https.png)

LAN内からAirsonicを利用できるようになりました。
次は最後のステップとしてAirsonic Podへの接続をhttps化し、インターネットに公開します。

### DNS設定
ルータのグローバルIPと自身の所有するドメイン(以降 `example.com`と表記)を紐づけるAレコードをDNSに追加します。
筆者は無料のダイナミックDNSサービスを使っています。

### ルータの設定
クラスターエンドポイントIP(192.168.0.XXX)の80番と443番ポートを開放します。

> この時点で http://example.com としてAirsonicにアクセスできるようになります。

### cert-manager

TLS証明書や発行者をk8sリソースとして管理可能にする[cert-manager](https://cert-manager.io/docs/)を導入します。

```sh
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

### Issuer

証明書発行者(Let's Encrypt)をk8sクラスタに追加します。
Let's Encryptの本番環境で試行錯誤しているとレート制限に引っかかる可能性があるため、検証用にステージング環境も追加します。
`spec.acme.solvers.http01.ingress.class` の値は環境に応じて設定してください。


<details><summary><font color="blue">issuer.yaml▼</font></summary><div>

```yaml:issuer.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: myaddress@example.com # UPDATE HERE
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: public # UPDATE HERE
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: myaddress@example.com # UPDATE HERE
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: public # UPDATE HERE
```

</div></details>

### Ingressを更新

Ingressをhttps対応に書き換えます。先ほど作成したIngressを削除し、以下の設定で再作成します。

```sh
$ kubectl delete ingress airsonic-http
$ kubectl create -f airsonic-ingress.yaml
```

<details><summary><font color="blue">airsonic-ingress.yaml▼</font></summary><div>

```yaml:airsonic-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airsonic
  namespace: default
  annotations:  
    cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
      - example.com
    secretName: tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: airsonic
            port: 
              number: 4040
```

</div></details>

Ingressを作成すると証明書リクエストが実施されます。リクエストは CertificateRequest としてk8sリソースになっています。

```sh
$ kubectl get cr
NAME        APPROVED   DENIED   READY   ISSUER              REQUESTOR                                         AGE
tls-xxxxx   True                True    letsencrypt-staging system:serviceaccount:cert-manager:cert-manager   1m
```

`APPROVED: True` になると証明書がREADYになります。

```sh
$ kubectl get cert
NAME   READY   SECRET   AGE
tls    True    tls      1m
```

> しばらく待っても証明書が取得できない場合は`kubectl describe cr` や `kubectl describe order` で原因を調査します。
> `example.com:80` へのチャレンジによって認証されるため、ルーターの設定などが原因になりがちです。

Let's Encryptステージング環境で正しく動作する事を確認したら、issuerを本番に切り替えて再作成します。

```yaml:airsonic-ingress.yaml
...
-   cert-manager.io/issuer: "letsencrypt-staging"
+   cert-manager.io/issuer: "letsencrypt-prod"
...
```

```sh
$ kubectl get ingress
NAME      CLASS    HOSTS           ADDRESS     PORTS     AGE
airsonic  public   example.com     127.0.0.1   80, 443   1m
```

> 証明書の期限が近づくと自動で更新されます。

### Airsonic設定ファイルの編集

ChromeからAirsonic Web UIにhttpsでアクセスすると、一部要素がMixed Contentsでブロックされてしまいます。
AirsonicのPodに入り、`X-Forwarded-For` を有効化する設定を追記してPodを再起動します。

```sh
$ kubectl exec -it airsonic-xxxxxxx bash
# echo "server.use-forward-headers=true" >> /config/airsonic.properties
# exit
$ kubectl rollout restart deployments/airsonic
```

以上でインターネットから https://example.com でAirsonicに接続できるようになります。

# おわりに

AirsonicはSubsonicというプロダクトからフォークされたもので、Subsonic APIに対応する様々なクライアントを利用することができます。
筆者はAndroidでは `Ultrasonic`、iOSでは `iSub` というアプリを利用しています。

<img src="/images/airsonic/mobile.jpg" width="240">

---

サンプルコードは最小限の実装になっていますので、実運用では環境に応じて冗長化やセキュリティ面の設定などを追加してください。
