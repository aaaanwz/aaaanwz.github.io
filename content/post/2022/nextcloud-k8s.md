---
title: "NextCloudのパーミッションエラー回避"
date: 2022-11-29
categories:
- Kubernetes
---

こんな感じでNextCloudをk8sにデプロイするとします。


<div class="toc"><details><summary accesskey="c">nextcloud.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
        - name: nextcloud
          image: nextcloud
          env:
          - name: NEXTCLOUD_DATA_DIR
            value: /data
          ports:
            - containerPort: 80
              protocol: TCP
          volumeMounts:
          - name: myvolume
            subPath: config
            mountPath: /var/www/html
          - name: myvolume
            subPath: data
            mountPath: /data
      volumes:
        - name: myvolume
          persistentVolumeClaim:
            claimName: myvolume
      securityContext:
        fsGroup: 33
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: default
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
  selector:
    app: nextcloud
  clusterIP: None
```

</details></div>


すると初回ログイン後に以下のようなエラーが出てきます。

```
データディレクトリ (/data) は他のユーザーも閲覧する事ができます。
ディレクトリが他のユーザーから見えないように、パーミッションを 0770 に変更してください。
```

基盤側の都合でパーミッションを変えられないケースの回避策(無理矢理)を紹介します。


a. `/var/www/html` をPersistentVolume (ex. `/config`) に永続化している場合

PersistentVolume (`hdd1`) を適当なインスタンスにマウントし、以下のファイルをエディタで編集します。

`/config/lib/private/legacy/OC_Util.php`

```php
...
	public static function checkDataDirectoryPermissions($dataDirectory) {
		if (\OC::$server->getConfig()->getSystemValue('check_data_directory_permissions', true) === false) {
			return  [];
		}

		$perms = substr(decoct(@fileperms($dataDirectory)), -3);
-   	if (substr($perms, -1) !== '0') {
+       if (substr($perms, -1) !== '7') {
- 		chmod($dataDirectory, 0770);
+ 		chmod($dataDirectory, 0777);
...
```

b. `/var/www/html` を永続化していない場合

Deploymentのlifecycle.postStartでパッチしてしまいます。

nextcloud.yaml

```yaml
...
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "sed -i -e 's/xxx/XXX/g' /var/www/html/lib/private/legacy/OC_Util.php "]
...
```

書いておいてなんですが、全くもっておすすめできる方法ではありません。