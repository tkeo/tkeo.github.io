---
layout: post
title: "rdbtoolsを使ってredisのデータの内訳を調べた"
date: 2017-03-08 01:06:23 +0900
comments: true
categories: redis 
---
最近redisのデータ量が増えすぎて、スワップを使い始めている状態になってしまった。
メモリ使用量は7.3GBでキーの数は1400万ぐらい。
スケールアップを検討しつつも、余計なデータがないか軽く調査。
以前にもやったことがあるけど忘れるので後のためにメモとして残しておく。

まずrdbファイルを取得する。[バックアップのエクスポート](http://docs.aws.amazon.com/ja_jp/AmazonElastiCache/latest/UserGuide/Snapshots.Exporting.html)で今日のバックアップをS3にエクスポートした。1.8GBぐらい。

ファイルがでかいのでAWS上で作業する。
本番ウェブサーバーのコピーを使って適当に立てたインスタンスに[rdbtools](https://github.com/sripathikrishnan/redis-rdb-tools)をインストール。

```
$ sudo pip install rdbtools
Downloading/unpacking rdbtools
  Downloading rdbtools-0.1.9.tar.gz
  Running setup.py (path:/tmp/pip_build_root/rdbtools/setup.py) egg_info for package rdbtools
    warning: no files found matching 'README.textile'
Downloading/unpacking redis (from rdbtools)
  Downloading redis-2.10.5-py2.py3-none-any.whl (60kB): 60kB downloaded
Installing collected packages: rdbtools, redis
  Running setup.py install for rdbtools
    warning: no files found matching 'README.textile'
    Installing redis-memory-for-key script to /usr/bin
    Installing redis-profiler script to /usr/bin
    Installing rdb script to /usr/bin
Successfully installed rdbtools redis
Cleaning up…
```

これでOKなはずだったが・・・

```
$ rdb -c memory dump-2017-03-06-0001.rdb -f redis-memory-2017-03-06.csv
Traceback (most recent call last):
  File "/usr/bin/rdb", line 5, in <module>
    from pkg_resources import load_entry_point
  File "/usr/lib/python2.6/site-packages/pkg_resources.py", line 2655, in <module>
    working_set.require(__requires__)
  File "/usr/lib/python2.6/site-packages/pkg_resources.py", line 648, in require
    needed = self.resolve(parse_requirements(requirements))
  File "/usr/lib/python2.6/site-packages/pkg_resources.py", line 546, in resolve
    raise DistributionNotFound(req)
pkg_resources.DistributionNotFound: redis
```

あれ？

```
$ pip list | grep redis
redis (2.10.5)
```

？？

以前やったときはこんなことにはならなかったが？？？

よく見るとどうやら最新は9日前にリリースされたバージョンで、前回使ったのは古いやつだったらしい。

```
$ sudo pip uninstall rdbtools
$ sudo pip install rdbtools==0.1.8
```

これでとりあえず動いた。pythonのバージョンの問題な気がするが、よくわからんので放置。

CSVの生成には20分ぐらいかかった。
次は出力されたCSVを加工していく。

```
database,type,key,size_in_bytes,encoding,num_elements,len_largest_element
0,string,"user:6665644:ip_address",104,string,13,13
0,string,"user:5568100:logged_in_time",96,string,8,8
0,string,"user:7540191:ip_address",104,string,13,13
0,string,"user:3428265:ip_address",104,string,12,12
0,string,"user:2176673:logged_in_time",96,string,8,8
...
```

出力されたCSVの中身はこんな感じ。
redis-objects gem使ってるとこういうキーのやつがいっぱいできちゃうよなー。
まとめやすいようにidっぽい部分を雑に置換する。

```
$ cat redis-memory-2017-03-06.csv | ruby -ne '_,_,key,size = $_.split(","); puts [key.gsub(/:\d+/, ":*"), size].join(",")' > tmp.csv
```

```
key,size_in_bytes
"user:*:ip_address",104
"user:*:logged_in_time",96
"user:*:ip_address",104
"user:*:ip_address",104
"user:*:logged_in_time",96
...
```

こいつを合計するスクリプトをざっと書いた。

```
$ cat redis-summary.rb
#!/usr/bin/env ruby

count = Hash.new {|h,k| h[k] = 0 }
sum = Hash.new {|h,k| h[k] = 0 }

while line = $stdin.gets
  key,size = line.split(",")
  count[key] += 1
  sum[key] += size.to_i
end

count.keys.each do |key|
  puts [key, count[key], sum[key]].join(",")
end
```

```
"user:*:ip_address",5537264,583765440
"user:*:logged_in_time",5538922,531736432
"room:*:ready_members",25817,3654184
"user_deck:*:total_power_cache",548302,57306072
"room:*:status_value",175756,19190496
...
```

余計なものがたくさんありそうなことはわかったけど、こいつらを消してもまだ足りなさそうな気がするなあ。

## 追記
定期的に削除する/できるようにする前のデータが残っていたのでまとめて削除した。
実装に手を入れたタイミングで消したつもりだった…。

![graph](/images/screenshot-2017-03-10.png)
