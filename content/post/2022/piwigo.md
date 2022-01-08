---
title: "Kubernetesに写真サーバーを構築する"
date: 2022-01-04
draft: false
categories:
- kubernetes
- 自宅サーバー
---

2021年6月1日からGoogle Photoの容量が無制限ではなくなり、無料枠は15GBに制限されてしまいました。  
完全に[音楽サーバーを構築した話]({{< ref "/post/2021/airsonic.md" >}})の二番煎じですが、自宅k8sに写真サーバーを構築してそちらに移行する事にしました。

## デプロイ
self-hostedな写真サーバーで最もメジャーなプロダクトは[Piwigo](https://piwigo.org/)の模様。  
サクっとyamlを書いてデプロイします。

<div class="toc"><details><summary accesskey="c">deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: piwigo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: piwigo
  template:
    metadata:
      labels:
        app: piwigo
    spec:
      containers:
        - name: piwigo
          image: lscr.io/linuxserver/piwigo
          env:
          - name: TZ
            value: Asia/Tokyo
          - name: PUID
            value: "1000"
          - name: PGID
            value: "1000"
          ports:
            - containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: 80
          volumeMounts:
          - name: myvolume
            subPath: piwigo/config
            mountPath: /config
          - name: myvolume
            subPath: piwigo/gallery
            mountPath: /gallery
      volumes:
        - name: myvolume
          persistentVolumeClaim:
            claimName: myvolume
---
apiVersion: v1
kind: Service
metadata:
  name: piwigo
  namespace: default
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
  selector:
    app: piwigo
  clusterIP: None
```

</details></div>

別途MySQLも必要なので、[Kubernetes公式ドキュメントのサンプルコード](https://kubernetes.io/ja/docs/tasks/run-application/run-replicated-stateful-application/#mysql%E3%82%92%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%99%E3%82%8B)あたりを参考にしてデプロイしておきます。

## 写真の追加

画像データをmyvolumeの `piwigo/gallery/galleries/{アルバム名}/{サブアルバム名}...` 以下に追加し、Web UIのAdminメニューから `Synchronize`を実行すればアルバムが自動で作成されます。  

この際、Simulationというdry-run的な機能が用意されており便利です。ファイル名にはスペースや全角文字が使えない等の制約がありますが、[Presync AutoRename](https://piwigo.org/ext/extension_view.php?eid=902)というプラグインを使うとある程度は自動でリネームできて便利です。  
  
もしくは以下のワンライナーでもOK。

```sh
$ find . -name "*{置換したい文字列}*" | rename 's/{置換したい文字列}/{置換先文字列}/g'
```

## プラグイン

画像を表示する際にデフォルトでは解像度がかなり落とされるので、[Automatic Size](https://piwigo.org/ext/extension_view.php?eid=702)プラグインを導入しました。  

> 他に良さげなプラグインがあれば随時追記予定

---

モバイルアプリもあり、Google Photoからの移行はスムーズに完了しました。  
スマホからの自動同期が無い点だけが不満ですが、容量を気にせず突っ込めるメリットは大きいです。

クラウドサービスからオンプレへの回帰が進み、自宅サーバーがどんどんミッションクリティカルになってきた...