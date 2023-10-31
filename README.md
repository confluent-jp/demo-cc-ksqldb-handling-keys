# ストリーム処理でKafka Topicを操る with ksqlDB
## はじめに
Kafka TopicはKey:Valueというレコード構成になっており、KeyはTopicのメタデータにあたる。その為通常ストリーム処理の対象はValueでありKeyに対して操作は行わない。

一方、このKeyを何かしらの理由で取得したり、処理条件の一部として指定する必要性が出る場合も多い。これにはSTREAMの定義やKSQLのクエリで操作する事ができる。

## Step 0: ストリーム処理全体像
![ストリーム処理フロー](/assets/images/stream-process-flow.png)
テスト用データはDataGen Source Connectorを利用して作成。データタイプはClickstreamとし、HTTP Response Codeを元に処理を切り分ける。

以下の3パターンのKeyの扱い方を検証する：
- Keyの無いSTREAMの作成
- KeyのあるSTREAMの作成
- Keyがあり、ValueにもKeyをコピーしたSTREAの作成

## Step 1: テストデータ用DataGen Connectorの設定
Clickstream用のDataGen Source Connectorを作成する。

メニューの**Connectors**から**Sample Data**を選択してConnectorを作成する。この際レコードフォーマットを```JSON```、テンプレートは```Clickstream```を選択する。
![DataGen Connectorの設定](/assets/images/creating-datagen-connector.png) ```datagen-topic```というTopicを新たに作成して関連付ける。
結果として生成されるConnectorのコンフィグサンプルは以下の通り：
```json
{
  "connector.class": "DatagenSource",
  "name": "DatagenSourceConnector_0",
  "kafka.auth.mode": "KAFKA_API_KEY",
  "kafka.api.key": "MUZO2GTWQTTXMZK3",
  "kafka.api.secret": "**************",
  "kafka.topic": "datagen-topic",
  "schema.context.name": "default",
  "output.data.format": "JSON",
  "json.output.decimal.format": "BASE64",
  "quickstart": "CLICKSTREAM",
  "max.interval": "1000",
  "tasks.max": "1"
}
```

## Step 2.1: Keyless STREAMの作成
通常```CREATE STREAM```で作成するSTREAMはKeyを含まない。
```sql
CREATE STREAM ClickstreamWithoutKey (
    IP VARCHAR,
    USERID INT,
    REMOTE_USER VARCHAR,
    TIME VARCHAR,
    _TIME INT,
    REQUEST VARCHAR,
    STATUS VARCHAR,
    BYTES VARCHAR,
    REFERRER VARCHAR,
    AGENT VARCHAR
) WITH (
KAFKA_TOPIC='datagen-topic', VALUE_FORMAT='JSON');
```

## Step 2.2: STREAM処理 - Keyless
2.1で作成したSTREAMから作成されるパイプラインにはそのスタートからKeyが設定されていない為その後もKeyは空のままとなる。
```sql
CREATE STREAM EventsWithoutKey
WITH (KAFKA_TOPIC='404events', VALUE_FORMAT='JSON')
AS SELECT
    IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM ClickstreamWithoutKey
WHERE STATUS = '404'
EMIT CHANGES;
```
結果として出力されるTopic(```404events```)の中身は以下の通り：
![Keyless - Key](/assets/images/keyless-key.png)
![Keyless - Value](/assets/images/keyless-value.png)
Topicから取得された```IP```はそのままValueの一部に指定されている。Keyには何も設定されていない。

## Step 3.1 STREAM wiht Keyの作成
```CREATE STREAM```でSTREAMを作成する際、対象フィールドに```Key```と指定するとそのフィールドを生成TopicのKeyに指定出来る。
```sql
CREATE STREAM ClickstreamWithKey (
    IP VARCHAR Key,
    USERID INT,
    REMOTE_USER VARCHAR,
    TIME VARCHAR,
    _TIME INT,
    REQUEST VARCHAR,
    STATUS VARCHAR,
    BYTES VARCHAR,
    REFERRER VARCHAR,
    AGENT VARCHAR
) WITH (
KAFKA_TOPIC='datagen-topic', VALUE_FORMAT='JSON');
```

## Step 3.2: STREAM処理 - with Key
Keyless(Step2.2)と全く同じ構文を利用した場合、今度はKeyに値が設定される。変わって```IP```フィールドはKeyとなった為Valueには出てこない。
```sql
CREATE STREAM EventsWithKey
WITH (KAFKA_TOPIC='200events', VALUE_FORMAT='JSON')
AS SELECT
    IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM ClickstreamWithKey
WHERE STATUS = '200'
EMIT CHANGES;
```
結果として出力されるTopic(```200events```)の中身は以下の通り：
![WithKey - Key](/assets/images/withkey-key.png)
![WithKey - Value](/assets/images/withkey-value.png)

## Step 3.3: STREAM処理 - with Key and Value
Step 3.2で生成されたSTREAMでは```IP```をKeyとして指定した為Valueからは除外されている。このKeyをValueにも持って来たい場合、CSAS(```CREATE STREAM AS SELECT```)構文にて細工が必要となる。
```sql
CREATE STREAM EventsWithKeyAndValue
WITH (KAFKA_TOPIC='302events', VALUE_FORMAT='JSON')
AS SELECT
    IP AS ROWKEY,
    AS_VALUE(IP) AS IP,
    USERID,
    _TIME TIME_IN_INT,
    STATUS,
    BYTES
FROM ClickstreamWithKey
WHERE STATUS = '302'
EMIT CHANGES;
```
結果として出力されるTopic(```302events```)の中身は以下の通り：
![WithKeyAndValue - Key](/assets/images/withkeyvalue-key.png)
![WithKeyAndValue - Value](/assets/images/withkeyvalue-value.png)
