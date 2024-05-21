# ER(Error Rate)計算を継承する場
## 2024/5/21
編集：m1 江本城太郎  
研究室で統一した実験結果になると良いよねというお話になったのでとりあえずでやってみる．

## 前提
[`jiwer`](https://github.com/jitsi/jiwer)モジュールを使用する．  
[`RapidFuzz`](https://github.com/rapidfuzz/RapidFuzz)という文字列のマッチングを取るモジュールをベースとして作成された誤り率計算用のモジュールとなっている．  
筆者は`Hugging Face`の`transformers`モジュールで音声認識モデルの開発を行っているが，  
モデル学習中のログを取る際にこれが使用されていたので手元で誤り率計算を行う際に用いている．  
Nvidiaの`NeMo`(https://github.com/NVIDIA/NeMo)でもモデル学習中のロギングやモデル評価時に使用されているみたい．  
pythonバージョン`3.8`以上で動作保証．

### pip
```terminal
$ pip install jiwer
```

### conda
一応`conda-forge`にも落ちてるっぽい．  
ただ`conda-forge`版は使用したことがないので`pip`版の使用をおすすめします．
```
$ conda install jiwer -c conda-forge
```

## usage
これだけ．([公式usageページ](https://jitsi.github.io/jiwer/usage/)から引用)
### code
```python
import jiwer

output = jiwer.process_words(
    ['short one here', 'quite a bit of longer sentence'],
    ['shoe order one', 'quite bit of an even longest sentence here'],
)

print(jiwer.visualize_alignment(output))
```
### result
```
sentence 1
REF: **** short one here
HYP: shoe order one ****
        I     S        D

sentence 2
REF: quite a bit of ** ****  longer sentence ****
HYP: quite * bit of an even longest sentence here
           D         I    I       S             I

number of sentences: 2
substitutions=2 deletions=2 insertions=4 hits=5

mer=61.54%
wil=74.75%
wip=25.25%
wer=88.89%
```

細かく解説していく．
```python
output = jiwer.process_words(
    ['short one here', 'quite a bit of longer sentence'],
    ['shoe order one', 'quite bit of an even longest sentence here'],
)
```

`process_words`メソッドには`string`もしくは`list`を2つ渡す．  
`list`を渡した場合はリスト中の各文章に対して個別に処理を行い，最後にまとめて計算してくれる．めちゃ便利．  
明示的な指定なしにリストを渡す場合には，
```python  
jiwer.process_words(reference_list, prediction_list)
```
のように`正解`，`評価対象`の順番で渡すこと．  
この後，`visualize_alignment()`関数の引数に先程の`output`を渡すことで誤りのアライメントを可視化しながら誤り率の表示が可能である．  
単純に誤り率が算出されるだけでなく，置換，削除，挿入の各誤りに対しても内訳が算出されているので個別に誤り率を計算することも可能である．

なお，`process_characters()`メソッドを用いることで`文字誤り率(CER)`も算出することが可能である．
```python
import jiwer

char_output = jiwer.process_characters(
    ['short one here', 'quite a bit of longer sentence'],
    ['shoe order one', 'quite bit of an even longest sentence here'],
)

print(jiwer.visualize_alignment(output))
```
### result
```
sentence 1
REF: short o*ne* here
HYP: shoe* order on*e
        SD  IS I SSD 

sentence 2
REF: quite a bit of ********longe*r sentenc*****e
HYP: quite **bit of an even longest sentence here
           DD       IIIIIIII     IS        IIIII 

number of sentences: 2
substitutions=5 deletions=4 insertions=16 hits=35

cer=56.82%
```
アライメントの可視化も行えている．  
なお，2文目の最後に関しては距離計算の関係で`sentence`の最後の`e`が`here`の最後の`e`として取られている． 
極端な例であるため，実利用して結果が悪い場合にはこれが効いている可能性もあるかもしれない．  
2バイト文字（ひらがな）とかは文字の幅が違うのでアライメントはほとんど意味がないことに注意．

ちなみに各値（置換誤りとか）は以下の書式で数字を参照できるので意外と便利かも．
```python
error_sub = output.substitutions
error_del = output.deletions
error_ins = output.insertions
hit_char  = output.hits
```

[研究室Github](https://github.com/kitaoka-lab/)内の[VAD_ADR_emoto](https://github.com/kitaoka-lab/VAD_ASR_emoto)のどこかのブランチに`jiwer`を用いてモデル評価を行っているコードがあるので具体的な処理の流れなど参考にしたい場合はそちらを参照してください．