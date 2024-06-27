# はじめに
これまで BETAリリースだった Oracle Database のベクトルデータベース機能 AI Vector Search を搭載したバージョンが先日ようやく正式リリースとなりました。その名は Oracle Database 23ai です。これまであらゆるワークロードを取り入れてきた Oracle Database ですが、晴れてベクトルデータベースのカテゴリに仲間入りです。

Oracle Databaseは沢山の先進的な機能を取り込んだデータベースですが、このリリースは名前からわかる通り、著しい早さで拡大しているAI市場を意識したバージョンとなります。とりわけ昨今のAI市場の話題の中心はやはり大規模言語モデル。本記事もそれに倣って、ベクトルデータベースとしてのOracle Database 23aiを使ったシンプルなRAG構成によるテキスト生成の実装サンプルをご紹介したいと思います。

※ RAGとは何ですか？という方はこちらの記事をご参照ください。

https://qiita.com/ksonoda/items/ba6d7b913fc744db3d79

# RAGの構成とシナリオ

### RAGの構成
本記事でご紹介するRAGの構成と処理フローの概要が下図になります。テキスト生成モデル、Rerankモデル、ベクトルデータベースを使った典型的なタイプのRAG構成です。LangChainやLlamaIndexなどのオーケストレーションツールを使用しないフルスクラッチのRAGとなります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/ffaaf9a4-f6a9-9cba-5cdf-8e20d4b079c4.png)


構成に利用するサービスは以下の通り。

- 大規模言語モデル：OCI Generative AI Service(Command R/Command R Plus)、番外編でCohere(Command R Plus)
- Rerank モデル：Cohere(rerank-multilingual-v3.0)
- ベクトルデータベース: Oracle Database 23ai Free
- ドキュメントデータのベクトル化に利用するモデル : Oracle Cloud Generative AI Service(embed-multilingual-v3.0)

### ドキュメントデータ
ベクトルデータベースにロードするドキュメントデータは下図のような内容のPDFファイルです。テキストの内容としては架空の製品であるロケットエンジンOraBoosterの説明文章です。企業内のデータを想定し、テキストの内容としては、存在しない製品かつ完全な創作文章、ということでLLMが学習しているはずのないデータということになります。後の手順でこちらのPDFファイルをベクトルデータベースにロードします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/76dc0197-cd00-ac65-5f4d-0f4a59a18391.png)

### シナリオ
本記事では下記のようなシナリオでテキスト生成を行い、LLMが学習していないデータ(上述のPDFの内容)に関する質問に対してRAG構成でうまく回答できることを確認します。また、非RAG構成とのテキスト生成結果を比較し、RAGの有効性を確認します。

1. 架空の製品OraBoosterについて質問をします。
2. LLMはこの製品の知識を持ち合わせていないため、ベクトルデータベースに類似検索を行い、ヒントとなるテキストデータ(チャンクテキスト)を複数検索します。下記のコードでは類似度の高いテキストデータを5つ検索するコードにしています。
3. 類似検索処理で得られた5つのテキストをRerankモデルで再スコアリングし、その結果から類似度の高いトップ3のテキストデータを取り出します。
4. Rerankモデルで取り出したテキストデータ3つと、もともとのプロンプトのテキストデータをテキスト生成モデルに入力し、OraBoosterの説明テキストが生成されていることを確認します。
5. ベクトルデータベースへの類似検索処理をスキップした状態(つまり、RAGではなく単にLLMにクエリしている状態)でテキスト生成を行い、上記「4」で生成されたテキストと比較しRAGの有効性を確認します。

※ 実装は興味ないので結果だけ知りたいですという方は「テキスト生成を実行」の章をご参照ください。


# 実装
下記の流れで実装してゆきます。

1. データベースインスタンスとデータベースの作成
2. ドキュメントファイル(PDF)をデータベースにロードし、ベクトル化 
3. RAGの実装


### データベースインスタンスとデータベースの作成
まずは、ベクトルデータベースとなるOracle Databaseを準備します。下記のチュートリアルに沿ってComputeインスタンスにOracle Database 23ai Freeをインストールします。

https://oracle-japan.github.io/ocitutorials/ai-vector-search/ai-vector102-23aifree-install/

データベースインストール後、下記の流れでユーザー作成、その他の設定を行います。

日本語表示のため、下記環境変数を定義します。

```sql
export NLS_LANG=Japanese_Japan.AL32UTF8
```

コンテナデータベース(CDB)内のプラガブル・データベース(PDB)にsysユーザーでsysdbaとして接続します。

```sql
sqlplus /nolog
CONN sys@freepdb1 as sysdba
```

DBユーザー(docuser)を作成し、必要な権限を付与します。

```sql
drop user docuser cascade; 
grant connect, ctxapp, unlimited tablespace, create credential, create procedure, create table to docuser identified by docuser;
grant execute on sys.dmutil_lib to docuser; 
grant create mining model to docuser; 

BEGIN
DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*', 
    ace => xs$ace_type(
        privilege_list => xs$name_list('connect'),
        principal_name => 'docuser', 
        principal_type => xs_acl.ptype_db)); 
END; 
/
```

インスタンスに配置したPDFドキュメントを読み込むためにディレクトリオブジェクトを作成します。こちらのOSのディレクトリに上述したPDFファイル(rocket.pdf)を配置してあります。


```sql
create directory VEC_DUMP as '/home/oracle/data/vec_dump';
grant read, write on directory VEC_DUMP to docuser; 
commit;
```

Oracle Databaseに接続します。

```sql
sqlplus /nolog
CONN docuser@freepdb1
```

SQL*Plusの表示を調整します。

```sql
SET ECHO ON 
SET FEEDBACK 1 
SET NUMWIDTH 10
SET LINESIZE 80 
SET TRIMSPOOL ON 
SET TAB OFF 
SET PAGESIZE 10000 
SET LONG 10000
```

### ドキュメントデータをデータベースにロードし、ベクトルデータに変換
Oracle Database 23aiにはドキュメントデータをデータベースにロードし、ベクトル化するための下記3つのプロシージャがあります。この3つのプロシージャを実行するだけで、PDFやWordファイルなど様々なドキュメントデータを簡単にベクトルデータベースに取り込むことができるようになっています。

- UTL_TO_TEXT : PDFファイルなどを構造解析し、テキストに変換(つまりドキュメントローダーです。)
- UTL_TO_CHUNK : 変換されたテキストデータをチャンクに分割(つまりテキストスプリッター(チャンカー)です。)
- UTL_TO_EMBEDDINGS : チャンクテキストをベクトルに変換(プロシージャの中で埋め込みモデルを指定しチャンクテキストをベクトルに変換します。)

流れとしては下図のように3つのプロシージャを順番に実行するだけです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/73fe64ac-c5ab-7d99-eccc-7b3c66d98acb.png)


以降、順番通りに処理を実行します。まず、表(documentation_tab)を作成し、その表にPDFドキュメントを格納します。

```sql
drop table documentation_tab purge; 
CREATE TABLE documentation_tab (id number, data blob);
INSERT INTO documentation_tab values(1, to_blob(bfilename('VEC_DUMP', 'rocket.pdf'))); 
commit;
```

UTL_TO_TEXTを実行してPDFをテキスト形式に変換します。

```sql
SELECT dbms_vector_chain.utl_to_text(dt.data) 
from documentation_tab dt;
```

UTL_TO_CHUNKSを実行して、テキストの全文がどのようにチャンクに分割されるかを下記SQLで確認します。今回は固定長で分割するので下記SQLでチャンクテキストを確認しながら max と overlap の値を調整します。

```sql
SELECT ct.* 
from documentation_tab dt,
    dbms_vector_chain.utl_to_chunks(
        dbms_vector_chain.utl_to_text(dt.data), 
        json('{"max": "100", "overlap": "10%", "language": "JAPANESE", "normalize": "all", "split": "sentence"}')
    ) ct;
```

最終的にチャンクテキストをベクトルに変換するためにOracle CloudのGenerative AI Serviceの埋め込みモデルを利用します。このサービスを利用するにはAPIキーでの認証が必要になりますから、APIキーをCredentialとしてデータベースに登録する処理を下記手順で行います。

まず、下記シェルコマンドを実行して、APIキーの生成で取得したkey_fileの文字列を取得します。

```bash
awk '/^--/{ next } { printf "%s", $0 } END { printf "\n" }' ~/.oci/oci_api_key.pem | tr -d '\n'
```

上記で取得した文字列をprivate_keyに入力し、APIキーの生成で取得したuser_ocid、tenancy_ocid、fingerprintおよびcompartment_ocidを入力して実行します。

```sql
exec dbms_vector.drop_credential('OCI_CRED'); 

declare 
jo json_object_t; 
begin 

jo := json_object_t(); 
jo.put('user_ocid', 'user ocid value'); 
jo.put('tenancy_ocid', 'tenancy ocid value'); 
jo.put('compartment_ocid', 'compartment ocid value'); 
jo.put('private_key', 'private key value'); 
jo.put('fingerprint', 'fingerprint value'); dbms_output.put_line(jo.to_string); 
dbms_vector.create_credential(
    credential_name => 'OCI_CRED', 
    params => json(jo.to_string)); 
end; 
/
```

OCI GenAIサービスにアクセスするためのパラメータを設定します。

```sql
var embed_genai_params clob; 
exec :embed_genai_params := '{"provider": "ocigenai", "credential_name": "OCI_CRED", "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText", "model": "cohere.embed-multilingual-v3.0"}';
```

上記の設定が動作するかOCI GenAIサービスへのアクセスして確認してみます。

```sql
select et.* from dbms_vector_chain.utl_to_embeddings(
    'こんにちは', 
    json(:embed_genai_params)
) et;
```

下記のように、入力したテキスト「こんにちは」のベクトル値が取得できていれば成功です。

```sql
COLUMN_VALUE 
--------------------------------------------------------
{"embed_id":"1","embed_data":"こんにちは","embed_vector":"[0.0035 934448,0.028701782,0.031051636,-0.001415
... 
1行が選択されました。
```

ここまでの手順でチャンク作成とベクトル化の準備ができました。
実際にUTL_TO_EMBEDDINGSを実行して、チャンクテキストをベクトルに変換します。

下記のSQLでは上述したUTL_TO_CHUNKSで分割したチャンクのテキストデータをUTL_TO_EMBEDDINGSでベクトル化しその値を保存する表を作成しています。

```sql
CREATE TABLE doc_chunks AS
WITH t_chunk AS (
  SELECT 
    dt.id AS doc_id, 
    et.chunk_id AS embed_id, 
    et.chunk_data AS embed_data
  FROM
    documentation_tab dt,
    dbms_vector_chain.utl_to_chunks(
      dbms_vector_chain.utl_to_text(dt.data), 
      JSON('{"max": "100", "overlap": "10%", "language": "JAPANESE", "normalize": "all", "split": "sentence"}')
    ) t, 
    JSON_TABLE(
      t.column_value, 
      '$[*]' COLUMNS (
        chunk_id NUMBER PATH '$.chunk_id', 
        chunk_data VARCHAR2(4000) PATH '$.chunk_data'
      )
    ) et
  WHERE 
    dt.id = 1
),
t_embed AS (
  SELECT 
    dt.id AS doc_id, 
    ROWNUM AS embed_id, 
    to_vector(t.column_value) AS embed_vector
  FROM
    documentation_tab dt,
    dbms_vector_chain.utl_to_embeddings(
      dbms_vector_chain.utl_to_chunks(
        dbms_vector_chain.utl_to_text(dt.data), 
        JSON('{"max": "100", "overlap": "10%", "language": "JAPANESE", "normalize": "all", "split": "sentence"}')
      ), 
      JSON('{
        "provider": "ocigenai", 
        "credential_name": "OCI_CRED", 
        "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText", 
        "model": "cohere.embed-multilingual-v3.0"
      }')
    ) t
  WHERE 
    dt.id = 1
)
SELECT 
  t_chunk.doc_id AS doc_id, 
  t_chunk.embed_id AS embed_id, 
  t_chunk.embed_data AS embed_data, 
  t_embed.embed_vector AS embed_vector
FROM 
  t_chunk
JOIN 
  t_embed 
ON 
  t_chunk.doc_id = t_embed.doc_id 
  AND t_chunk.embed_id = t_embed.embed_id;
```

PDFファイルがベクトル化されている表に対して、「OraBoosterとは何ですか？」という文章で類似検索を実行し、類似度の高い上位5つをクエリしてみます。

```sql
SELECT doc_id, embed_id, embed_data 
FROM doc_chunks 
ORDER BY vector_distance(embed_vector , (
    SELECT to_vector(et.embed_vector) embed_vector 
    FROM dbms_vector_chain.utl_to_embeddings('OraBoosterとは何ですか？', JSON(:embed_genai_params)) t, 
        JSON_TABLE ( t.column_value, '$[*]' COLUMNS ( 
            embed_id NUMBER PATH '$.embed_id', 
            embed_data VARCHAR2 ( 4000 ) PATH '$.embed_data', 
            embed_vector CLOB PATH '$.embed_vector' 
        )) et
    ), COSINE) 
FETCH EXACT FIRST 5 ROWS ONLY;
```

下記のような出力が得られれば成功です。

```sql
    DOC_ID   EMBED_ID
---------- ----------
EMBED_DATA
--------------------------------------------------------------------------------
         1          1
PowerPoint Presentation

当社が開発したロケットエンジンであるOraBoosterは、次世代の宇宙探査を支える先進的
な推進技術の

象徴です。その独自の設計は、高性能と革新性を融合させ、人類の宇宙進出を加速させる
ための革命的

な一歩となります。

         1          5
また、ハイパーフォトン・ジャイロスコープが搭載されており、極めて高い精度でロケッ
トの姿勢を維

持し、目標を追跡します。
....
....
....
....
....
```

ここまでの手順で、PDFファイルのテキストデータをベクトルデータベースにベクトルデータとしてロードする部分が完了しました。類似検索が動作することも確認できました。RAGの中で最も重要なパーツが構成できた状態です。

### RAGの実装
ここからがRAGの実装です。これ以降の処理はPythonで実装します。データベースインスタンスにPython環境を実装していただいてもいいですし、オンプレミスやクラウド環境で処理しても大丈夫です。

プロンプトを入力した際に下記の流れでコードを繋ぎ、RAGのフローを実装します。

- プロンプトのテキストをベクトル化し、ベクトルデータベースで類似検索実行しトップ5のチャンクテキストを抽出
- その結果をリランクモデルで再スコアリングし精度の高いトップ3のチャンクテキストを抽出
- リランク結果のトップ3のチャンクテキストともともとのプロンプトをLLMに入力しテキストを生成

まずはデータベースの接続とカーソルを定義します。

```python
import oracledb

conn = oracledb.connect(user="docuser", password="docuser", dsn="localhost/freepdb1")
cur = conn.cursor()
```

ここからベクトルデータベースでの類似検索を行います。


まずは、query(プロンプトの文章)を定義します。このqueryはベクトルデータベースの類似検索の時と、テキスト生成モデルに入力する時に使います。

```sql
query = 'OraBoosterとは何ですか？'
```

このプロンプトでベクトルデータベースに類似検索を実行します。SQLは上述したベクトルデータベースの類似検索の動作確認時に実行したものとほぼ同じです。(SQL最後の行"FETCH FIRST 5 ROWS ONLY"の件数も変数にしてもいいと思います。)

```python
sql_query = """
    SELECT doc_id, embed_id, embed_data 
    FROM doc_chunks 
    ORDER BY vector_distance(
        embed_vector, 
        (
            SELECT TO_VECTOR(et.embed_vector) AS embed_vector 
            FROM DBMS_VECTOR_CHAIN.UTL_TO_EMBEDDINGS(
                :query, 
                JSON('{"provider": "ocigenai", "credential_name": "OCI_CRED", "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText", "model": "cohere.embed-multilingual-v3.0"}')
            ) t, 
            JSON_TABLE (
                t.column_value, 
                '$[*]' COLUMNS (
                    embed_id NUMBER PATH '$.embed_id', 
                    embed_data VARCHAR2(4000) PATH '$.embed_data', 
                    embed_vector CLOB PATH '$.embed_vector'
                )
            ) et
        )
    , COSINE) 
    FETCH FIRST 5 ROWS ONLY
"""

cur.execute(sql_query, {'query': query})
rows = cur.fetchall()
rows
```

出力としては、下記のように類似検索の結果「OraBoosterとは何ですか？」というテキストのベクトル値と類似度の高い5件のチャンクテキストが取得できています。(取得したチャンクテキスト内に不要な文字列や改行コードが入っていますが、本来は前処理でちゃんとクレンジングします。)


```python
[(1,1,'PowerPoint Presentation\n\n当社が開発したロケットエンジンであるOraBoosterは、次世代の宇宙探査を支える先進的な推進技術の\n\n象徴です。その独自の設計は、高性能と革新性を融合させ、人類の宇宙進出を加速させるための革命的\n\nな一歩となります。'),
 (1,5,'また、ハイパーフォトン・ジャイロスコープが搭載されており、極めて高い精度でロケットの姿勢を維\n\n持し、目標を追跡します。'),
 (1,6,'これにより、長時間にわたる宇宙飛行中でも安定した飛行軌道を維持し、\n\nミッションの成功を確保します。\n\nさらに、バイオニック・リアクション・レスポンダーが統合されています。'),
 (1,8,'総じて、この新開発のロケットエンジンは、革新的な技術と未来志向の設計によって、宇宙探査の新た\n\nな時代を切り開くことでしょう。その高い性能と信頼性は、人類の夢を実現するための力強い支援とな\n\nることでしょう。'),
 (1,2,'このエンジンの核となるのは、量子ダイナミックス・プラズマ・ブースターです。このブースターは、\n\n量子力学の原理に基づいてプラズマを生成し、超高速で加速させます。')]
```

少し見にくいのでdataframeに変換して表示してみます。

```python
import pandas as pd

df = pd.DataFrame(rows)
print(df)
```

0の列はドキュメントIDです。今回は一つのPDFファイルをロードしましたが、複数のPDFファイルをロードした場合は1以外のドキュメントIDがここに表れる可能性があります。1の列がチャンクIDで2の列がチャンクに分割されたテキストです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/abc1a8f5-51b8-fd3b-f8e1-3dc7d19b2ab8.png)



「OraBoosterとは何ですか？」というテキストで検索しているので、チャンクID 8(新開発のロケットエンジン)がが上位に来てほしい感じがしますが、残念ながら類似度が下位に計算されているという状況です。これがセマンティック検索の難しさですよね。従って、このランキングを少しでも人間の感覚に近づけるべく、この後にRerankのモデルで再スコアリングを行います。


※因みに、ベクトルの類似検索自体は、テキスト生成モデルの入力処理とは全く異なりますから、「〇〇は何ですか？」のように質問のようなテキストにする必要はありません。


この結果セットにはドキュメントIDやチャンクIDといった重要なデータが含まれています。これらを使うことによって、最終的なテキスト生成とのきにどのドキュメントのどのチャンクテキストを参照したのかという情報をユーザーに見えるようにすることもできます。が、今回はシンプルに基本のコードの骨格を理解したいのでこの部分を省こうと思います。下記のコードでテキストの部分だけを取り出します。

```python
results = [row[2] for row in rows]
```

ここでCohereのRerankモデルの登場です。下記のようにもともとのクエリテキスト(query)と類似検索の結果のチャンクテキスト(results)をRerankモデルに入力し、再スコアリング後の類似度トップ3のチャンクテキストを取り出します。


```python
import cohere
co = cohere.Client("xxxxxxxxx")

# query = 'OraBoosterとは何ですか？'
results_rerank = co.rerank(
    model = "rerank-multilingual-v3.0", 
    # queryはプロンプトテキストであり、類似検索の検索テキストです。
    query = query, 
    # resultsは類似検索結果。今回の場合上述した5つのチャンクテキストが入ったリストです。
    documents = results, 
    return_documents = True, 
    top_n = 3
)

print(results_rerank)
```

下記のように3つのチャンクテキストやスコアなど様々な情報が確認できます。チャンクテキスト以外のデータも重要ではありますが、今回はシンプルにテキストデータのみを使います。

```python
RerankResponse(id='35c5e447-3163-4ab6-8a44-adecc6bbca0f', results=[RerankResponseResultsItem(document=RerankResponseResultsItemDocument(text='PowerPoint Presentation\n\n当社が開発したロケットエンジンであるOraBoosterは、次世代の宇宙探査を支える先進的な推進技術の\n\n象徴です。その独自の設計は、高性能と革新性を融合させ、人類の宇宙進出を加速させるための革命的\n\nな一歩となります。'), index=0, relevance_score=0.9998378), RerankResponseResultsItem(document=RerankResponseResultsItemDocument(text='これにより、長時間にわたる宇宙飛行中でも安定した飛行軌道を維持し、\n\nミッションの成功を確保します。\n\nさらに、バイオニック・リアクション・レスポンダーが統合されています。'), index=2, relevance_score=0.01302049), RerankResponseResultsItem(document=RerankResponseResultsItemDocument(text='総じて、この新開発のロケットエンジンは、革新的な技術と未来志向の設計によって、宇宙探査の新た\n\nな時代を切り開くことでしょう。その高い性能と信頼性は、人類の夢を実現するための力強い支援とな\n\nることでしょう。'), index=3, relevance_score=0.011115014)], meta=ApiMeta(api_version=ApiMetaApiVersion(version='1', is_deprecated=None, is_experimental=None), billed_units=ApiMetaBilledUnits(input_tokens=None, output_tokens=None, search_units=1.0, classifications=None), warnings=None))
```

上記の出力だと見にくいので少し整形します。テキストデータだけを取り出し、改行コードを削除してみます。

```python
contents = []
for result in results_rerank.results:
    contents.append(result.document.text)
    
def remove_newline(text):
    return text.replace('\n', '')

contents_list = [remove_newline(text) for text in contents]
print(contents_list)
```

結果、再ランキングされたTop3のチャンクテキストが下記の3つ。このリランクの処理により、期待していた「新開発のロケットエンジン」と「量子ダイナミックス・プラズマ・ブースター」のチャンクテキストがTop3にランクインしてくれました。やはりRerankモデルはなかなか使えます。

```python
['PowerPoint Presentation\n\n当社が開発したロケットエンジンであるOraBoosterは、次世代の宇宙探査を支える先進的な推進技術の\n\n象徴です。その独自の設計は、高性能と革新性を融合させ、人類の宇宙進出を加速させるための革命的\n\nな一歩となります。',
 'これにより、長時間にわたる宇宙飛行中でも安定した飛行軌道を維持し、\n\nミッションの成功を確保します。\n\nさらに、バイオニック・リアクション・レスポンダーが統合されています。',
 '総じて、この新開発のロケットエンジンは、革新的な技術と未来志向の設計によって、宇宙探査の新た\n\nな時代を切り開くことでしょう。その高い性能と信頼性は、人類の夢を実現するための力強い支援とな\n\nることでしょう。']
```

上記のリストはstring型ですが、Cohereのテキスト生成のLLMはjson型しか入力できないため下記のように型変換を行います。

```python
json_contents_list = []
for content in contents_list:
    json_contents_list.append({"text": content})

print(json_contents_list)
```

json型になりましたのでこのjson_contents_listをテキスト生成モデルの入力に使います。

```python
[{'text': 'これにより、長時間にわたる宇宙飛行中でも安定した飛行軌道を維持し、ミッションの成功を確保します。さらに、バイオニック・リアクション・レスポンダーが統合されています。'}, {'text': '総じて、この新開発のロケットエンジンは、革新的な技術と未来志向の設計によって、宇宙探査の新たな時代を切り開くことでしょう。その高い性能と信頼性は、人類の夢を実現するための力強い支援となることでしょう。'}, {'text': 'このエンジンの核となるのは、量子ダイナミックス・プラズマ・ブースターです。このブースターは、量子力学の原理に基づいてプラズマを生成し、超高速で加速させます。'}]
```
### テキスト生成を実行(Generative AI Service の Command RもしくはCommand R Plus)
ここまで処理した内容は以下の通りです。

1. PDFファイルをベクトル化しデータベースにロード
2. プロンプトの内容に沿った類似検索を実行し類似度の高いトップ5を取得
3. Rerankモデルによりトップ5のテキストデータを再スコアリングし、トップ3を取得

上記3のRerankingの結果のトップ3のチャンクテキストともともとのプロンプトをテキスト生成モデルに入力して最終的なテキスト生成を行います。このテキスト生成モデルに、まずはOCI Generative AI ServiceのCommand-Rを使ってみたいと思います。

まずはOCIの環境とGenerative AI Serviceのエンドポイントを定義します。

```python
import oci

compartment_id = "ocid1.compartment.oc1..xxxxxxxx2bijwewlq"
CONFIG_PROFILE = "DEFAULT"
config = oci.config.from_file('~/.oci/config', CONFIG_PROFILE)

# Service endpoint
endpoint = "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com"
```

次に、モデルとモデルのパラメータを定義します。下記セルの下2行のところ、chat_request.messageにqueryのテキストを、chat_request.documentsにベクトル類似検索およびreranking結果のチャンクテキストを入力しています。

```python
generative_ai_inference_client = oci.generative_ai_inference.GenerativeAiInferenceClient(config=config, service_endpoint=endpoint, retry_strategy=oci.retry.NoneRetryStrategy(), timeout=(10,240))

chat_detail = oci.generative_ai_inference.models.ChatDetails()
chat_request = oci.generative_ai_inference.models.CohereChatRequest()

# モデル(Command RもしくはCommand R Plus)の定義
# command-r のモデルを定義
# chat_detail.serving_mode = oci.generative_ai_inference.models.OnDemandServingMode(model_id="ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyawk6mgunzodenakhkuwxanvt6wo3jcpf72ln52dymk4wq")
# command-r-plusのモデルを定義
chat_detail.serving_mode = oci.generative_ai_inference.models.OnDemandServingMode(model_id="ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya7ozidbukxwtun4ocm4ngco2jukoaht5mygpgr6gq2lgq")
chat_detail.chat_request = chat_request
chat_detail.compartment_id = compartment_id

# モデル(Command RもしくはCommand R Plus)のパラメータの定義
chat_request.max_tokens = 600
chat_request.temperature = 1
chat_request.frequency_penalty = 0
chat_request.top_p = 0.75
chat_request.top_k = 0

# プロンプトの入力
chat_request.message = query

# 類似検索結果の入力
chat_request.documents = json_contents_list
```

モデルの定義が終わりましたのでテキスト生成を行い、結果を表示します。

```python
chat_response = generative_ai_inference_client.chat(chat_detail)
print(chat_response.data.chat_response.text)
```

出力としては以下の通り、Reranking結果(とベクトルデータベースの類似検索結果)を使って正しいテキストが生成されていることが分かります。

```python
OraBoosterは、先進的な推進技術を持つ当社開発のロケットエンジンです。その独自性と高性能さは人類の宇宙進出に革命をもたらすでしょう。長時間にわたる宇宙飛行中、安定した飛行軌道を維持し、ミッションの成功率を高めます。バイオニック・リアクション・レスポンダーが搭載されています。
```

ついでに、Reranking結果(とベクトルデータベースの類似検索結果)を使わないパターン、つまりRAGでない構成でテキスト生成をしてみます。下記のとおり、モデルの定義のコードのchat_request.documentsの部分をコメントアウトしてRAGでない状況を作ります。

```python
generative_ai_inference_client = oci.generative_ai_inference.GenerativeAiInferenceClient(config=config, service_endpoint=endpoint, retry_strategy=oci.retry.NoneRetryStrategy(), timeout=(10,240))

chat_detail = oci.generative_ai_inference.models.ChatDetails()
chat_request = oci.generative_ai_inference.models.CohereChatRequest()

# モデル(Command RもしくはCommand R Plus)の定義
# command-r のモデルを定義
# chat_detail.serving_mode = oci.generative_ai_inference.models.OnDemandServingMode(model_id="ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyawk6mgunzodenakhkuwxanvt6wo3jcpf72ln52dymk4wq")
# command-r-plusのモデルを定義
chat_detail.serving_mode = oci.generative_ai_inference.models.OnDemandServingMode(model_id="ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya7ozidbukxwtun4ocm4ngco2jukoaht5mygpgr6gq2lgq")
chat_detail.chat_request = chat_request
chat_detail.compartment_id = compartment_id

chat_request.max_tokens = 600
chat_request.temperature = 1
chat_request.frequency_penalty = 0
chat_request.top_p = 0.75
chat_request.top_k = 0

# プロンプトの入力
chat_request.message = query

# 類似検索結果の入力
# chat_request.documents = json_contents_list
```

この状態からテキスト訂正を行います。

```python
chat_response = generative_ai_inference_client.chat(chat_detail)
print(chat_response.data.chat_response.text)
```

すると、下記の通り、ハルシネーションが発生したテキストが生成されていることが分かります。

```python
OraBoosterは、口腔衛生と歯の健康に良い影響を与えることを目的とした口腔衛生デバイスです。オーラル・Bによって作られた、ソニック技術を使った口腔洗浄デバイスで、歯のクリーニングと消毒を助けると言われています。オーラル・ケアのルーティンを向上させるために設計され、歯科医によるクリーニングの当時に近づけることを目的としています。  طالオーバは、フッ化物洗浄やmassageモードなどの機能がいくつかあり、口腔衛生を改善するために使用できます。それはあなたの歯科衛生習慣に追加できるポータブルで便利なデバイスです。しかし、歯科医や歯科衛生士による専門的なクリーニングの代替とは見なされません。
```


### テキスト生成(Cohere社 Command-R-Plusのモデル)

番外編としてCohere社 Command-R-Plusのモデルでもテキスト生成をしてみたいと思います。下記コードのmessageがプロンプトのテキストデータ、documentsが類似検索とRerankにより得られたトップ3のチャンクテキストです。

```python
import cohere
co = cohere.Client("xxxxxxxxx")

response = co.chat(  
    model='command-r-plus',
    # queryはプロンプトテキスト。今回の場合は「OraBoosterとは何ですか？」です。
    message=query,
    # documentsにリランク後のチャンクテキスト3つを入力します。
    documents=json_contents_list
)

print(response.text)
```

実行してみると、ちゃんとベクトルデータベースのドキュメントを使ってテキストが生成されていることがわかります。

```python
'OraBoosterは、次世代の宇宙探査のための先進的な推進技術を備えた、当社が開発したロケットエンジンです。その革新的な技術と未来志向の設計により、宇宙探査の新たな時代を切り開くとともに、長時間の宇宙飛行中でも安定した飛行軌道を維持することができます。'
```

更に、RAGを構成せずにテキスト生成モデルだけに任せてテキスト生成をしてみます。LLMへの問い合わせをする下記コードの中で、ベクトルデータベースへの類似検索結果を指定する引数documentsの項目をコメントアウトしてみます。

```python
response = co.chat(  
    model='command-r-plus',
    message=query,
    # documentsをコメントアウトし、類似検索の結果を使わないように設定
    # documents=json_contents_list
)

print(response.text)
```

すると、結果は当然ハルシネーション満載のテキストが生成されます。

```python
'OraBooster は、歯の健康をサポートし、口腔内の環境を改善することを目的とした口腔衛生用品のブランドです。OraBooster の製品には、歯磨き粉、マウスウォッシュ、歯ブラシなどが含まれ、口腔内の細菌のバランスを整え、歯と歯茎の健康を促進するとされています。\n\nOraBooster の製品は、天然成分を使用しており、口腔内の pH バランスを最適化することでプラークや歯垢の形成を抑制し、歯肉炎や歯周病を予防する効果が期待できます。これらの製品は、歯のエナメル質を強化し、歯を白くする効果もあると言われています。\n\nOraBooster は、口腔内の健康が全身の健康と密接に関係しているという考えに基づいて開発されており、口腔内の環境を改善することで、より健康的な生活を送ることを目指しています。これらの製品は、歯科医や医療専門家と協力して開発されており、口腔衛生の維持に役立つ便利で効果的なソリューションを提供することを目的としています。'
```

このように比較するとRAGの有効性がよくわかりますね。

# さいごに
現在ベクトルデータベースの市場には数十ともいわれる製品が存在します。(たしかLangChainがサポートしているベクトルデータベースは70以上あったと思います。)そのような中でベクトルデータベースとしてOracle Databaseを使うメリットは何なのか？と考えてみると以下3つほどが思い浮かびます。

- Oracle Databaseは言うまでもなくRDBですので、基本的な処理はSQLで実行します。その点SQLに慣れ親しんでいるユーザーにとって扱いやすいという点は間違いないでしょう。しかも、SQLで実行できるこの基本処理が単純なベクトル検索だけではなく、ドキュメントローダー、テキストスプリッター(チャンカー)までカバーしています。また、昨今はRAGの精度改善にプロンプト変換と呼ばれる手法を使うケースがあります。LangChainやLlamaIndexにも実装されておりよく使われるテクニックですが AI Vector SearchでもUTL_TO_GENERATE_TEXT()というプロシージャで類似の処理ができるようになっています。その他のUTL_TO_XXX系は是非使いこなしてみてください。

- RDBがゆえに、PineconeやFaiss、Chromaといったベクトルネイティブなベクトルデータベースと異なり、表構造を利用した検索もやりやすいと思います。今回の記事ではドキュメント内のテキストデータをベクトル検索するだけのシンプルな処理サンプルをご紹介しましたが、実システムではこのような単純な非構造化データにプラスして、構造化データを組み合わせた検索のニーズもあると思います。そのような場合、表構造を生かし、更にSQLで検索できるRDBの仕組みは大いに有効だと感じます。皆さまの企業におかれましてもOracle Databaseを運用していただいているケースは多いかと思います。そのOracle Databaseの中にはビジネスの価値を生み出すデータがあり、それらを有効活用するためのベクトルデータベースを選定する場合、やはり同じOracle Databaseの親和性は間違いないでしょう。

- ベクトルデータベースではついついAI関連の新機能や、ベクトルデータのデータ処理関連の機能ばかりに目が行きがちですが、プロダクションシステムを作る場合は、やはりデータベースの選定をしているという意識を忘れないようにしたいものですね。いかにベクトルデータベースとはいえ結局は性能、可用性、運用性などの要件をベースに製品選定をすることにります。その点において、老舗のOracle Databaseはエンタープライズのデータベースとして堅牢な機能や仕組みを求めるユーザーに選ばれてきた実績があり、そのようなデータベースが今やクラウドで簡単にそして安価に利用できる点は大きなメリットだと思います。

と、多少オラクル贔屓に書いてしましましたが、少なからずご賛同いただける方もいらっしゃるのではないでしょうか。この機会にエンタープライズ向けのOracle Database 23aiとCohereで実装するエンタープライズRAGをOracle Cloudで動かしてみるのはいかがでしょうか。
