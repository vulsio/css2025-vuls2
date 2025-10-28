# 広範な脆弱性情報の統合管理と履歴追跡 @ コンピュータセキュリティシンポジウム2025

<div style="text-align:center">
  <img src="https://vuls.io/img/docs/vuls_logo.png">
</div>

## 資料

- 論文PDF: https://raw.githubusercontent.com/vulsio/css2025-vuls2/main/docs/css.pdf
- 発表スライドPDF: https://raw.githubusercontent.com/vulsio/css2025-vuls2/main/slide/css.pdf

cf.
- CSS 2025: https://www.iwsec.org/css/2025/
- OSSセキュリティ技術ワークショップ(OWS) 2025: https://www.iwsec.org/ows/2025/#ows2025_program

## 参考 URL

実装関係

- vuls-data-update
  - https://github.com/MaineK00n/vuls-data-update
  - raw, extracted の取得、変換ロジックや dotgit を扱うサブコマンドなど
- vuls2
  - https://github.com/MaineK00n/vuls2
  - データベース関係のコマンドや検知ロジック

GitHub Container Registry 関係

- vuls-data-db
  - https://github.com/vulsio/vuls-data-db/pkgs/container/vuls-data-db
  - raw, extracted の履歴管理されたデータ置き場
- vuls-nightly-db
  - https://github.com/vulsio/vuls-nightly-db/pkgs/container/vuls-nightly-db
  - データベース置き場
  - 過去バージョンは https://github.com/vulsio/vuls-nightly-db/pkgs/container/vuls-nightly-db/versions

その他

- VulsDB
  - https://cve.vuls.biz/
  - extracted データを元にした脆弱性情報を参照できるサイト


## 簡単な使い方

ここでは、論文の 3.4.1 節にある Red Hat 社脆弱性情報の過去履歴の追跡と、
データベースから脆弱性情報のスナップショットまでのたどり方、実際のコマンド付きで説明する。
コマンドラインツールおよびデータはすべてパブリックであり、読者は実際に試しながら読むことができる。

### 事前準備

Go のインストール:

- https://go.dev/doc/install に従ってインストールする

本論文で実装したツールのインストール:

```
% go install github.com/MaineK00n/vuls-data-update/cmd/vuls-data-update@nightly
% go install github.com/MaineK00n/vuls2/cmd/vuls@nightly
```

これにより、 `vuls-data-update` および `vuls` バイナリがインストールされる。

### 論文「3.4.1.1 OS パッケージに対するステータスの変化」で言及した CVE-2024-0229 の情報

まずは Red Hat の VEX フォーマットの脆弱性情報の取得する。
カレントディレクトリ直下に vuls-data-extracted-redhat-vex-rhel を持ってくる:

```
% vuls-data-update dotgit pull ghcr.io/vulsio/vuls-data-db:vuls-data-extracted-redhat-vex-rhel --dir .
2025/10/21 11:22:30 [INFO] Pull dotgit from ghcr.io/vulsio/vuls-data-db:vuls-data-extracted-redhat-vex-rhel

% ls -al ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel
total 12
drwxr-xr-x 3 shino shino 4096 Oct 21 11:22 ./
drwxr-xr-x 3 shino shino 4096 Oct 21 11:22 ../
drwxr-xr-x 7 shino shino 4096 Oct 21 11:23 .git/
```

このように、データは git 管理で、 `.git/` だけが存在する。
"git restore ." で working tree を復元することはできる。しかしディスク容量を食うので注意が必要である。
以下では restore せずに `.git/` の中を見ていく。

次に、CVE ID からファイルパスを特定する。
このために `vuls-data-update` の `dotgit find` サブコマンドを利用できる:

```
% vuls-data-update dotgit find ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel CVE-2024-0229
100644 blob 066c83ec7a71148f0c501f9c948e14ea99bb692b    76517   data/2024/CVE-2024-0229.json
```

ファイルパスが分かったので、まずは最新のファイルの中身を確認してみる:

```
% vuls-data-update dotgit cat ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel data/2024/CVE-2024-0229.json | head -10
{
        "id": "CVE-2024-0229",
        "advisories": [
                {
                        "content": {
                                "id": "RHSA-2024:0320",
                                "references": [
                                        {
                                                "source": "secalert@redhat.com",
                                                "url": "https://access.redhat.com/errata/RHSA-2024:0320"
```

次に、該当ファイルの変更履歴の確認してみる:

```
% vuls-data-update dotgit log ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel --pathspec data/2024/CVE-2024-0229.json | grep -E '(commit|Date:)' | head -10
commit 3b4035793c5a5ee070faa92212bb4c7e6c890efc
Date:   Wed Oct 15 17:15:04 2025 +0000
commit a3aa0db1a0b681829ba3394e135abc2cb811c575
Date:   Wed Oct 15 05:41:31 2025 +0000
commit aa4daa5e71e0dfb4e30de71202e782520dd7c677
Date:   Tue Oct 14 16:43:23 2025 +0000
commit dc1fd9a0c51c7891bc171847df4db65d2b5275fb
Date:   Tue Oct 14 05:38:43 2025 +0000
commit f3a74c85ae9ee2b45301c1f154cb7ec42fee2821
Date:   Sun Oct 12 08:40:43 2025 +0000
```

実際に「3.4.1.1 OS パッケージに対するステータスの変化」にある `8c5e6158b4` と `3e8d8d3bf5` の差分を表示する:

```
% git -C ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel diff 8c5e6158b4:data/2024/CVE-2024-0229.json 3e8d8d3bf5:data/2024/CVE-2024-0229.json
diff --git a/data/2024/CVE-2024-0229.json b/data/2024/CVE-2024-0229.json
index ff2d7030545..3ffe8f531ed 100644
--- a/data/2024/CVE-2024-0229.json
+++ b/data/2024/CVE-2024-0229.json
@@ -199,25 +199,41 @@
 					}
 				],
 				"published": "2024-01-16T00:00:00Z",
-				"modified": "2025-01-07T00:57:25Z"
+				"modified": "2025-07-09T08:20:13Z"
 			},
 			"segments": [
 				{
 					"ecosystem": "redhat:6",
 					"tag": "rhel-6-els:438841e3-9d81-edce-c802-90e38c1b085d"
 				},
[中略]
@@ -307,7 +307,7 @@
 									"vulnerable": true,
 									"fix_status": {
 										"class": "unfixed",
-										"vendor": "Out of support scope"
+										"vendor": "Affected"
 									},
 									"package": {
 										"type": "source",
[後略]
```

この出力から `fix_status` の変化が見て取れる。
差分の周辺の情報を見てから `redhat:6` の `tigervnc` に絞って見てみると、"Out of support scope" から "Affected" へ変化していることが分かる。

```
% vuls-data-update dotgit cat ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel --treeish 8c5e6158b4 data/2024/CVE-2024-0229.json | jq -M -c '.detections.[] | select( .ecosystem == "redhat:6" ) | .conditions.[] | .criteria.criterions.[] | select(.version.package.binary.name == "tigervnc" or .version.package.source.name == "tigervnc") | [.version.fix_status, .version.package]'
[{"class":"unfixed","vendor":"Out of support scope"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Out of support scope"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Out of support scope"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Out of support scope"},{"type":"source","source":{"name":"tigervnc"}}]

% vuls-data-update dotgit cat ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel --treeish 3e8d8d3bf5 data/2024/CVE-2024-0229.json | jq -M -c '.detections.[] | select( .ecosystem == "redhat:6" ) | .conditions.[] | .criteria.criterions.[] | select(.version.package.binary.name == "tigervnc" or .version.package.source.name == "tigervnc") | [.version.fix_status, .version.package]'
[{"class":"unfixed","vendor":"Affected"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Affected"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Affected"},{"type":"source","source":{"name":"tigervnc"}}]
[{"class":"unfixed","vendor":"Affected"},{"type":"source","source":{"name":"tigervnc"}}]
```

このように JSON 文字列の diff だけからこの情報にたどり着くまでには、人間が周辺を調査する必要がある。
これをより簡単にすることは今後の課題のひとつである。

### 論文「3.4.1.2 特定の CVE ID に対する情報が消滅する事例」で言及した CVE-2024-24791 の情報

もし、あるCVE IDがデータソースから削除された場合、履歴では次のように確認することができる。

```
% vuls-data-update dotgit diff tree --pathspec data/2024/CVE-2024-24791.json ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel 708ff21dc6 e4787b5085
diff --git a/data/2024/CVE-2024-24791.json b/data/2024/CVE-2024-24791.json
deleted file mode 100644
index 663aedd4577..00000000000
--- a/data/2024/CVE-2024-24791.json
+++ /dev/null
@@ -1,4321 +0,0 @@
-{
-	"id": "CVE-2024-24791",
...
-							{
-								"type": "version",
-								"version": {
-									"vulnerable": true,
-									"fix_status": {
-										"class": "fixed"
-									},
-									"package": {
-										"type": "binary",
-										"binary": {
-											"name": "podman-tests",
-											"architectures": [
-												"aarch64",
-												"ppc64le",
-												"s390x",
-												"x86_64"
-											]
-										}
-									},
-									"affected": {
-										"type": "rpm",
-										"range": [
-											{
-												"lt": "2:5.2.2-1.el9"
-											}
-										],
-										"fixed": [
-											"2:5.2.2-1.el9"
-										]
-									}
-								}
-							}
-						]
-					},
-					"tag": "rhel-9-including-unpatched:e1cd6b61-06c1-e8ae-d404-0a2e70fccc55"
-				}
-			]
-		}
-	],
-	"data_source": {
-		"id": "redhat-vex",
-		"raws": [
-			"vuls-data-raw-redhat-repository-to-cpe/repository-to-cpe.json",
-			"vuls-data-raw-redhat-vex/2024/CVE-2024-24791.json"
-		]
-	}
-}
```

また、削除されたCVE IDが再度追加された場合も、引き続き追跡可能である。

```
% vuls-data-update dotgit log --pathspec data/2024/CVE-2024-24791.json ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel/
...
commit 9d4f3266739277edec70a213071bb7867713cc5b
Author: GitHub Action <action@github.com>
Date:   Fri Aug 8 15:26:19 2025 +0000

    update

commit e4787b5085615f1cedeef7492ed4bf414ec949c1
Author: GitHub Action <action@github.com>
Date:   Fri Jul 18 03:47:59 2025 +0000

    update

commit 4b0894c677db2677f86da49c7218c67448e94ca9
Author: GitHub Action <action@github.com>
Date:   Tue Jul 15 03:27:29 2025 +0000

    update
...

% vuls-data-update dotgit diff tree --pathspec data/2024/CVE-2024-24791.json ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel e4787b5085 9d4f326673
diff --git a/data/2024/CVE-2024-24791.json b/data/2024/CVE-2024-24791.json
new file mode 100644
index 00000000000..efe2972cfe2
--- /dev/null
+++ b/data/2024/CVE-2024-24791.json
@@ -0,0 +1,4321 @@
+{
+	"id": "CVE-2024-24791",

+							{
+								"type": "version",
+								"version": {
+									"vulnerable": true,
+									"fix_status": {
+										"class": "fixed"
+									},
+									"package": {
+										"type": "binary",
+										"binary": {
+											"name": "podman-tests",
+											"architectures": [
+												"aarch64",
+												"ppc64le",
+												"s390x",
+												"x86_64"
+											]
+										}
+									},
+									"affected": {
+										"type": "rpm",
+										"range": [
+											{
+												"lt": "2:5.2.2-1.el9"
+											}
+										],
+										"fixed": [
+											"2:5.2.2-1.el9"
+										]
+									}
+								}
+							}
+						]
+					},
+					"tag": "rhel-9-including-unpatched:e1cd6b61-06c1-e8ae-d404-0a2e70fccc55"
+				}
+			]
+		}
+	],
+	"data_source": {
+		"id": "redhat-vex",
+		"raws": [
+			"vuls-data-raw-redhat-repository-to-cpe/repository-to-cpe.json",
+			"vuls-data-raw-redhat-vex/2024/CVE-2024-24791.json"
+		]
+	}
+}
```

### 論文「3.4.1.3 Red Hat 10 に対する情報のみが消滅する事例」で言及した CVE-2024-43420 の情報

特定のエコシステムに対する検知情報全体が削除された例を示す。
commit: 3e8d8d3bf5 で Red Hat 10 で CVE-2024-43420 を検知する検知情報が削除されたことが、履歴から分かる。

```
% vuls-data-update dotgit diff tree --pathspec data/2024/CVE-2024-43420.json ghcr.io/vulsio/vuls-data-db/vuls-data-extracted-redhat-vex-rhel 8c5e6158b4 3e8d8d3bf5
diff --git a/data/2024/CVE-2024-43420.json b/data/2024/CVE-2024-43420.json
index 1e57028da92..ed3a107162e 100644
--- a/data/2024/CVE-2024-43420.json
+++ b/data/2024/CVE-2024-43420.json
@@ -77,13 +77,9 @@
 					}
 				],
 				"published": "2024-01-01T00:00:00Z",
-				"modified": "2025-07-03T19:54:37Z"
+				"modified": "2025-07-10T04:59:46Z"
 			},
 			"segments": [
-				{
-					"ecosystem": "redhat:10",
-					"tag": "rhel-10:5e2b9810-f4da-f914-b943-5f9f7f1d3bb4"
-				},
 				{
 					"ecosystem": "redhat:6",
 					"tag": "rhel-6-els:56b6b192-0a7c-8e3b-ac70-cb1b87ff4f45"
@@ -112,35 +108,6 @@
 		}
 	],
 	"detections": [
-		{
-			"ecosystem": "redhat:10",
-			"conditions": [
-				{
-					"criteria": {
-						"operator": "OR",
-						"criterions": [
-							{
-								"type": "version",
-								"version": {
-									"vulnerable": true,
-									"fix_status": {
-										"class": "unfixed",
-										"vendor": "Fix deferred"
-									},
-									"package": {
-										"type": "source",
-										"source": {
-											"name": "microcode_ctl"
-										}
-									}
-								}
-							}
-						]
-					},
-					"tag": "rhel-10:5e2b9810-f4da-f914-b943-5f9f7f1d3bb4"
-				}
-			]
-		},
 		{
 			"ecosystem": "redhat:6",
 			"conditions": [
```

### データベースから脆弱性情報のスナップショットまでのたどり方

まずは最新版のデータベースを取得する:

```
% vuls db fetch --repository ghcr.io/vulsio/vuls-nightly-db:0 --dbpath ./vuls-latest.db
2025/10/21 12:31:24 INFO Fetch vuls.db repository=ghcr.io/vulsio/vuls-nightly-db:0
⠹ fetching (1.9 GB, 55 MB/s) [34s]

% ls -alh ./vuls-latest.db
-rw-r--r-- 1 shino shino 1.8G Oct 21 12:32 ./vuls-latest.db
```

データベース内には、脆弱性の情報が入っている。たとえば上で見た CVE-2024-0229 の情報は以下のように参照できる:

```
% vuls db search vulnerability CVE-2024-0229 --dbpath ./vuls-latest.db | jq . | head -20
2025/10/21 12:34:10 INFO Get Metadata
2025/10/21 12:34:10 INFO Get Vulnerability Data queries=[CVE-2024-0229]
{
  "id": "CVE-2024-0229",
  "advisories": [
    {
      "id": "RHSA-2024:0320",
      "contents": {
        "redhat-vex": {
          "CVE-2023-6816": [
            {
              "content": {
                "id": "RHSA-2024:0320",
                "references": [
                  {
                    "source": "secalert@redhat.com",
                    "url": "https://access.redhat.com/errata/RHSA-2024:0320"
                  }
                ],
                "published": "2024-01-22T13:53:02Z"
              },
              "segments": [
```

まず、データベースにはそれ自身のメタデータが付与されている:

```
% vuls db search metadata --dbpath ./vuls-latest.db | jq -M .
2025/10/21 14:41:24 INFO Get Metadata
{
  "schema_version": 0,
  "created_by": "vuls v0.0.1-alpha.0.20251003074613-517cb812e035",
  "last_modified": "2025-10-21T01:19:35.916413982Z",
  "digest": "sha256:7dbf3bcf8d76151beef3a908a016f9fbadebba7c63fbc54a3dae77a6609dd5d0",
  "downloaded": "2025-10-21T05:41:16.868754065Z"
}
```

この `"digest"` がこのデータベースを一意に指定する ID となっている。
GitHub container registry に入っているので https://github.com/vulsio/vuls-nightly-db/pkgs/container/vuls-nightly-db/versions で一覧が見れる。

過去の検知結果を調査する際には、その検知に使ったデータベースの digest がわかれば完全に同じデータベースを用意できる。
具体的に `sha256:8c670d143634bc8c0438bd610d1243942b047e8560e5ff3ce48c3a7a0f24b4fe` が必要な場合は digest を指定して以下のように取得する:

```
% vuls db fetch --repository ghcr.io/vulsio/vuls-nightly-db@sha256:8c670d143634bc8c0438bd610d1243942b047e8560e5ff3ce48c3a7a0f24b4fe --dbpath ./vuls-8c67.db
2025/10/21 14:46:37 INFO Fetch vuls.db repository=ghcr.io/vulsio/vuls-nightly-db@sha256:8c670d143634bc8c0438bd610d1243942b047e8560e5ff3ce48c3a7a0f24b4fe
⠏ fetching (1.9 GB, 55 MB/s) [33s]

% ls -alh ./vuls-8c67.db
-rw-r--r-- 1 shino shino 1.8G Oct 21 14:47 ./vuls-8c67.db

% vuls db search metadata --dbpath ./vuls-8c67.db | jq -M .
2025/10/21 14:47:57 INFO Get Metadata
{
  "schema_version": 0,
  "created_by": "vuls v0.0.1-alpha.0.20251003074613-517cb812e035",
  "last_modified": "2025-10-14T12:58:13.619475736Z",
  "digest": "sha256:8c670d143634bc8c0438bd610d1243942b047e8560e5ff3ce48c3a7a0f24b4fe",
  "downloaded": "2025-10-21T05:47:14.163203208Z"
}
```

データベースが用意できたら、脆弱性情報の元となったデータに関する情報を `search datasources` で参照できる。

```
% vuls db search datasources --dbpath ./vuls-latest.db | jq -M . | head -20
2025/10/21 12:36:50 INFO Get Metadata
2025/10/21 12:36:50 INFO Get DataSources
[
  {
    "id": "alma-errata",
    "name": "AlmaLinux Errata",
    "raw": [
      {
        "url": "ghcr.io/vulsio/vuls-data-db:vuls-data-raw-alma-errata",
        "commit": "fb939f8061e26cbb3a365d15b1ad2d0b2743c0fe",
        "date": "2025-10-18T02:28:50Z"
      }
    ],
    "extracted": {
      "url": "ghcr.io/vulsio/vuls-data-db:vuls-data-extracted-alma-errata",
      "commit": "61431ba6bb6609cfb01d13af572ec283145c90ef",
      "date": "2025-10-18T03:31:43Z"
    }
  },
  {
    "id": "alpine-secdb",
    "name": "Alpine Linux Security Fixes Database",
```

この情報から AlmaLinux の脆弱性検知データは
- `ghcr.io/vulsio/vuls-data-db` の `vuls-data-raw-alma-errata` タグに raw (生データに近い形)があり、
- `ghcr.io/vulsio/vuls-data-db` の `vuls-data-extracted-alma-errata` タグに extracted (統合された構造)がある
ことがわかる。

`ghcr.io/vulsio/vuls-data-db` の情報は https://github.com/vulsio/vuls-data-db/pkgs/container/vuls-data-db/versions や、oras ツールでも参照可能である。

上記の Red Hat の VEX データと同様に、 `vuls-data-extracted-alma-errata` タグを取得すると `.git/` が入ったファイルが得られる。
その git リポジトリのコミットハッシュが `"commit"` にかかれているため、ピンポイントでデータの出どころが分かる。

先ほど取得した `vuls-8c67.db` からも raw, extracted のコミット時点を特定できる。
つまり、過去の検知時点での raw, extracted のデータを調査可能であり、またその時点のデータと別の時刻でのデータの差分が調査可能となる。

## 付録

```bibtex
@InProceedings{css2025-vuls,
  title = {広範な脆弱性情報の統合管理と履歴追跡},
  etitle = {Integrated Management and Historical Tracking of Diverse Vulnerability Information},
  author = {中岡 典弘, 篠原 俊一},
  yomi = {Norihiro Nakaoka, Shunichi Shinohara},
  booktitle = {コンピュータセキュリティシンポジウム2025論文集},
  pages = {xxx--xxx},
  year = {2025},
  month = {10},
  note = {開催地: 岡山コンベンションセンター},
  annote = {https://www.iwsec.org/css/2025/program.html}
}
```

## Copyright

Copyright by Norihiro Nakaoka, Shunichi Shinohara and Future Corporation.
