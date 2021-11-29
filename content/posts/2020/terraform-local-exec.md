---
title: "terraform非対応リソースをlocal-execで管理する"
date: 2020-09-14
draft: false
categories:
- terraform
---

terraformに対応していないクラウドリソースを `local-exec` を用いてterraform化してみます。
今回はBigQueryのユーザー定義関数(UDF)でやってみます。

# 実装

さて早速。

```terraform:main.tf
variable project{}

resource "null_resource" "bigquery-udf" { <- #1
  triggers = {
    query = "CREATE OR REPLACE FUNCTION my_dataset.TEST_FUNCTION(x INT64) AS (x + 1);" <- #2
  }
  provisioner "local-exec" { <- #3
    interpreter = ["bq", "query", "--use_legacy_sql=false", "--project_id=${var.project}"] <- #4
    command     = self.triggers.query
    on_failure  = fail <- #5
  }
  provisioner "local-exec" {
    when        = destroy <- #6
    interpreter = ["bq", "query", "--use_legacy_sql=false", "--project_id=${var.project}"]
    command     = "DROP FUNCTION IF EXISTS ${element(regex("FUNCTION\\s(.+?)[\\s\\(]", file("${path.module}/udf/${each.value}")), 0)}" <- #7
    on_failure  = fail
  }
}
```

# 解説

## `#1`
執筆時点では `google` provderはBigQueryのUDFに対応していないため、 `null_resource` として定義します。
https://www.terraform.io/docs/provisioners/null_resource.html

## `#2`
リソースの本体です。この部分が変更されるとresource updateとして扱われます。
https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource#argument-reference

## `#3`
リソースのcreate時の挙動を`local-exec` provisionerとして定義します。
https://www.terraform.io/docs/provisioners/local-exec.html

## `#4`
実際に実行するコマンドを定義します。
今回の例では `command` にSQL自身を、`interpreter` にはSQLを実行するための `bq` コマンドを記述します。
https://www.terraform.io/docs/provisioners/local-exec.html#argument-reference

## `#5`
コマンドの失敗時の挙動 (terraformコマンド自体を失敗させるかどうか) を設定します。
https://www.terraform.io/docs/provisioners/index.html#failure-behavior

## `#6`
when = destroyを宣言するとリソース削除時に実行するコマンドを定義できます。
https://www.terraform.io/docs/provisioners/index.html#destroy-time-provisioners

## `#7`
terraformとは関係ありませんが、リソース削除時はDDLから関数名を正規表現で抽出してDROPするようにしてみました。

---
複数のUDFを管理する場合は`provisioner`の定義が冗長になるので、`query`をvariableに定義して`for_each` を使う実装になるかと思います。
CI/CDパイプライン等のterraformコマンドの実行環境に`bq`コマンドを用意しなければならないデメリットがあるため、リソースをterraform管理にするメリットがそのコストを上回ると判断される場合には有用かと思います。

