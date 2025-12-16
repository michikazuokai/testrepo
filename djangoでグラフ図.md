<iframe id="google_ads_iframe_/4374287/blo_pc_com_6_3328_0_no_0" name="google_ads_iframe_/4374287/blo_pc_com_6_3328_0_no_0" title="3rd party ad content" width="336" height="280" scrolling="no" marginwidth="0" marginheight="0" frameborder="0" aria-label="Advertisement" tabindex="0" allow="private-state-token-redemption;attribution-reporting" data-load-complete="true" data-google-container-id="1" style="max-width: 100%; display: inline-block !important; border: 0px; vertical-align: bottom;"></iframe>

[2018-05-15](https://hideharaaws.hatenablog.com/archive/2018/05/15)

# [Django で Model の グラフ図を出力](https://hideharaaws.hatenablog.com/entry/2018/05/15/183533)

[Django](https://hideharaaws.hatenablog.com/archive/category/Django) [python](https://hideharaaws.hatenablog.com/archive/category/python)

本エントリーは、過去エントリー : [Django の ER図 出力 が 2ステップで出来た - AWS / PHP / Python ちょいメモ](http://hideharaaws.hatenablog.com/entry/2014/12/06/020322) の修正などを含めて記載しております。

[Django](http://d.hatena.ne.jp/keyword/Django) では、Modelの定義を行う事でデータベースは定義は自動的に作成されます。

[django](http://d.hatena.ne.jp/keyword/django)-extensions を使うことで、Modelからグラフ図を出力することができます （ER図とは少々違い、Modelの継承などが表現されてるので、[UML](http://d.hatena.ne.jp/keyword/UML)のクラス図に近いようです。でもリレーションの表現が甘い？このあたり詳しくない。。。）。内部的には、「[Django](http://d.hatena.ne.jp/keyword/Django) の Model定義 -> dot ファイル -> [Graphviz](http://d.hatena.ne.jp/keyword/Graphviz) で図にする」という手順を踏んでいるので、 dot ファイルの方が扱いやすい方は、画像出力じゃない方法もとれるようです。

見つけてしまえば、たった２ステップ。簡単ですね（開発者に感謝！）



#### 環境

- [Ubuntu](http://d.hatena.ne.jp/keyword/Ubuntu) 14.04
- [Python](http://d.hatena.ne.jp/keyword/Python) 3.5.3
- pip 9.0.1
- [Django](http://d.hatena.ne.jp/keyword/Django) 1.8.17

#### 手順

先述の環境が動作してる前提で、次のコマンドをたたき、pygraphvizなどのインストール環境を作る。

```
$ sudo apt-get install libgraphviz-dev graphviz pkg-config
```

その後、pip でメインのパッケージをインストール。

```
$ sudo pip3 install pygraphviz
$ sudo pip3 install pydotplus
$ sudo pip3 install django-extensions
```

上記により、次がインストールされました。

- pygraphviz-1.3.1
- pydotplus-2.0.2
- [django](http://d.hatena.ne.jp/keyword/django)-extensions-2.0.7


導入したい[Django](http://d.hatena.ne.jp/keyword/Django)プロジェクトの settings.py に Appを追加する。名前がアンダースコアなのに注意。

```python
INSTALLED_APPS = (
    ...
    'django_extensions',
)
```

#### Graph Model を使う

グラフ図を出力。 -a : all-applications , -g : group-models といったオプションです。

```
$ python manage.py graph_models -a -g -o graph-model.png
```

dot のみが必要な場合には -o なしで実行すればOK。

```
$ python3 manage.py graph_models auth
```

さらに特定のModelだけ含める場合

```
$ python3 manage.py graph_models auth -I User,Group
```

[Django](http://d.hatena.ne.jp/keyword/Django) の auth App のグラフ図を出力した例です。

```
$ python manage.py graph_models auth -g -o graph-model-auth.png
```

Proxy Model として作った Personal もきちんと表現されてる



![f:id:hidehara:20180515183300p:plain](https://cdn-ak.f.st-hatena.com/images/fotolife/h/hidehara/20180515/20180515183300.png)[django](http://d.hatena.ne.jp/keyword/django)-extentions の Graph models で出力



#### 出力オプション

ちなみに出力ファイル名のところを変えると 画像：[png](http://d.hatena.ne.jp/keyword/png), jpg の他、[ベクター](http://d.hatena.ne.jp/keyword/%A5%D9%A5%AF%A5%BF%A1%BC)：[svg](http://d.hatena.ne.jp/keyword/svg) , PDF：pdf , HTML:plain も出力できました。その他、こんな拡張子のファイルがだせるようです。

[Graphviz - Graph Visualization Software](http://www.graphviz.org/) でサポートするものなら、なんでもいけるのかな？便利ですね。

> Use one of: [canon](http://d.hatena.ne.jp/keyword/canon) cmap cmapx cmapx_np dot eps fig gd gd2 gif gv [imap](http://d.hatena.ne.jp/keyword/imap) [imap](http://d.hatena.ne.jp/keyword/imap)_np ismap jpe [jpeg](http://d.hatena.ne.jp/keyword/jpeg) jpg pdf pic plain plain-ext [png](http://d.hatena.ne.jp/keyword/png) pov ps [ps2](http://d.hatena.ne.jp/keyword/ps2) [svg](http://d.hatena.ne.jp/keyword/svg) svgz tk [vml](http://d.hatena.ne.jp/keyword/vml) vmlz [vrml](http://d.hatena.ne.jp/keyword/vrml) wbmp [x11](http://d.hatena.ne.jp/keyword/x11) xdot xdot1.2 xdot1.4 xlib


ヘルプは、manage.py help で参照できます。

```
$ python3 manage.py help graph_models
usage: manage.py graph_models [-h] [--version] [-v {0,1,2,3}]
                              [--settings SETTINGS] [--pythonpath PYTHONPATH]
                              [--traceback] [--no-color] [--pydot]
                              [--group-models] [--language LANGUAGE]
                              [--disable-fields] [--verbose-names]
                              [--exclude-models EXCLUDE_MODELS]
                              [--disable-sort-fields]
                              [--include-models INCLUDE_MODELS]
                              [--no-inheritance] [--json] [--all-applications]
                              [--output OUTPUTFILE] [--inheritance]
                              [--hide-relations-from-fields]
                              [--exclude-columns EXCLUDE_COLUMNS]
                              [--pygraphviz] [--layout LAYOUT]
                              [app_label [app_label ...]]

Creates a GraphViz dot file for the specified app names. You can pass multiple
app names and they will all be combined into a single model. Output is usually
directed to a dot file.

positional arguments:
  app_label

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit...
```

#### 参考

- [Welcome to the django-extensions documentation! — django-extensions 2.0.7 documentation](https://django-extensions.readthedocs.io/en/latest/)
- [若手プログラマー必読！５分で理解できるER図の書き方５ステップ](http://it-koala.com/entity-relationship-diagram-1897)
