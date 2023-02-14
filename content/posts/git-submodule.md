---
title: "git submoduleについての備忘録"
date: 2023-02-14T14:59:04+09:00
tags: ["備忘録", "git"]
draft: false
---
よく忘れる操作をまとめました。

なお各項目において、GitHub上の`cohsh/qeinput`（ローカルにおいては`./qeinput`）をサブモジュールの例として扱います。
#### 追加
```
git submodule add https://github.com/cohsh/qeinput.git
```

#### 削除
```
git submodule deinit -f qeinput
git rm -f qeinput
rm -rf .git/modules/qeinput
```

#### 更新
```
git submodule update --remote
```

#### サブモジュールを含むリポジトリの`clone`
`cohsh/qeinput`をサブモジュールとして含む`cohsh/q-e-template`を例とします。
```
git clone --recursive https://github.com/cohsh/q-e-template.git
```
