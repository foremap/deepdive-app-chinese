# 1. Introduction 

This project is a deepDive transaction example with Chinese Support. 

# 2. Prerequisite

* **DeepDive >= 0.8 stable** 

The DeepDive is only running on Linux . we can run bash script to install DeepDive online .

```bash
bash <(curl -fsSL git.io/getdeepdive)
```

Then , we can quickly install DeepDive by selecting the deepdive option:

```
### DeepDive installer for Linux
1) deepdive                   5) jupyter_notebook
2) deepdive_docker_sandbox    6) postgres
3) deepdive_example_notebook  7) run_deepdive_tests
4) deepdive_from_release      8) spouse_example
# Install what (enter to repeat options, a to see all, q to quit, or a number)? 1

```

open `~/.bashrc` file , add `export PATH=$PATH:$HOME/local/bin` .  run command `$ deepdive` 

![2018-10-14-10-57-54](http://www.xdpie.com/2018-10-14-10-57-54.png)




* **Postgresql 9.6**

We can select postgres from DeepDive's installer to install it and spin up an instance on you machine, or just run the following command:

``` bash 
bash <(curl -fsSL git.io/getdeepdive) postgres
```

* **CoreNlp model** 

For chinese support , we must download stanford-srparser-2014-10-23-models.jar and stanford-chinese-corenlp-2016-01-19-models.jar from stanford NLP website.

> https://nlp.stanford.edu/software/stanford-chinese-corenlp-2016-01-19-models.jar
> https://nlp.stanford.edu/software/stanford-srparser-2014-10-23-models.jar

Next, put these jar in `./udf/bazaar/parser/lib` , like this .

![2018-10-14-11-50-14](http://www.xdpie.com/2018-10-14-11-50-14.png)

Finally , we need to excute `$ sbt/sbt stage` to rebulid  (note: command in `parser` file ). 

* **Raw Input Data**

The raw input data include two parts , company's transaction data and announcements data。We can get more transaction from http://www.gtarsc.com/DataMarket/Index.

![2018-10-14-10-41-52](http://www.xdpie.com/2018-10-14-10-41-52.png)

In our project, we put some data in ./input/transaction_dbdata.csv file and articles.csv .  

# 3. Processing Raw Input Data 

## 3.1. Initialize Postgresql

DeepDive will store all data—input, intermediate, output, etc.—in a relational database. Currently, Postgres, Greenplum, and MySQL are supported; however, Greenplum or Postgres are strongly recommended. To set the location of this database, we need to configure a URL in the db.url file, e.g.:

`$ echo "postgresql://$USER@$HOSTNAME:5432/deepdive_spouse_$USER" >db.url`

## 3.2. Declare Schema
we need to declare the schema of this articles table in our `./app.ddlog` file; we add the following lines:

``` js
@source
articles(
	@key
	@distributed_by
    id      text,
    @searchable
    content text
).
```

``` js
@source
transaction_dbdata(
    @key
    company1_name text,
    @key
    company2_name text
).
```

Then we compile our application, as we must do whenever we change app.ddlog:

``` bash
$ deepdive compile
```

## 3.3. Loading Raw Input Data 
we tell DeepDive to execute the steps to load the articles table using the 'input/articles.csv' file 

``` bash
$ deepdive do transaction_dbdata &&  deepdive do articles
```
After that finishes, we can take a look at the loaded data using the following deepdive sql command, which enumerates the values for the id column of the articles table: 

``` bash 
$ deepdive sql "SELECT id FROM articles"
```

```
     id     
------------
 1201734370
 1201734454
 1201734455
 1201734457
 1201734458
 1201734460
 1201734461
 1201734464
 1201738707
 1201738753
 1201738764
 ...
 ```


# 4、 Processing 

Next, we'll use Stanford's CoreNLP natural language processing (NLP) system to add useful markups and structure to our input data. This step will split up our articles into sentences and their component tokens (roughly, the words). Additionally, we'll get lemmas (normalized word forms), part-of-speech (POS) tags, named entity recognition (NER) tags, and a dependency parse of the sentence. We declare the output schema of this step in `app.ddlog`:

``` js
@source
sentences(
	@key
    @distributed_by
    doc_id         text,
    @key
    sentence_index int,
    @searchable
    sentence_text  text,
    tokens         text[],
    lemmas         text[],
    pos_tags       text[],
    ner_tags       text[],
    doc_offsets    int[],
    dep_types      text[],
    dep_tokens     int[]
).

```

Next we declare a DDlog function which takes in the doc_id and content for an article and returns rows conforming to the sentences schema we just declared, using the user-defined function (UDF) in udf/nlp_markup.sh. This UDF is a Bash script which calls our own wrapper around CoreNLP. The CoreNLP library requires Java 8 to run.

```
function nlp_markup over (
        doc_id  text,
        content text
    ) returns rows like sentences
    implementation "udf/nlp_markup.sh" handles tsv lines.

```

Finally, we specify that this nlp_markup function should be run over each row from articles, and the output appended to sentences:

```
sentences += nlp_markup(doc_id, content) :-
    articles(doc_id, content).
```

Again, to execute, we compile and then run:


```
$ deepdive compile && deepdive do sentences
```

Now, if we take a look at a sample of the NLP markups, they will have split every article into some sentences that look like the following:

`$ deepdive sql "SELECT doc_id,sentence_index FROM sentences"
`

``` js

doc_id   | sentence_index 
------------+----------------
 1201734370 |              1
 1201734370 |              2
 1201734370 |              3
 1201734370 |              4
 1201734370 |              5
 1201734370 |              6
 1201734370 |              7
 1201734370 |              8
 1201734370 |              9
 1201734370 |             15
 1201734370 |             10
 1201734370 |             11
 1201734370 |             12
 1201734370 |             13
 1201734370 |             14
 1201734370 |             16
 1201734370 |             17
 1201734370 |             18
 1201734370 |             19
 1201734370 |             20
 1201734370 |             21
 1201734370 |             22
 1201734370 |             23
 1201734370 |             24
 1201734370 |             25
 1201734370 |             26
 1201734370 |             27


```

# 5、Extracting Data

## Extracting candidate relation mentions

Once again we first declare the schema:
``` Python
@extraction
company_mention(
	@key
    mention_id     text,
    @searchable
    mention_text   text,
    @distributed_by
    @references(relation="sentences", column="doc_id",         alias="appears_in")
    doc_id         text,
    @references(relation="sentences", column="doc_id",         alias="appears_in")
    sentence_index int,
    begin_index    int,
    end_index      int
).
```

We will be storing each company entity(ORG) as a row referencing a sentence with beginning and ending indexes. Again, we next declare a function that references a UDF and takes as input the sentence tokens and NER tags:

```
function map_company_mention over (
        doc_id         text,
        sentence_index int,
        tokens         text[],
        ner_tags       text[]
    ) returns rows like company_mention
    implementation "udf/map_company_mention.py" handles tsv lines.

company_mention += map_company_mention(
    doc_id, sentence_index, tokens, ner_tags
) :-
    sentences(doc_id, sentence_index, _, tokens, _, _, ner_tags, _, _, _).

```

```
$ deepdive compile && deepdive do company_mention
```

after excuted ,we can query data `$ deepdive sql "SELECT doc_id,mention_text FROM company_mention"`

```

doc_id   |                     mention_text                     
------------+------------------------------------------------------
 1201734370 | 郴州市城市建设投资发展集团有限公司
 1201734370 | 郴州市城市建设投资发展集团有限公司
 1201734370 | 郴州城投公司
 1201734370 | 郴州城投公司
 1201734370 | 市建设投资发展集团有限公司
 1201734370 | 郴州市城市建设投资发展集团有限公司
 1201734370 | 郴州城投公司
 1201734370 | 郴州城投公司
 1201734370 | 国家专项建设基金
 1201734370 | 郴州城投公司
 1201734370 | 郴州城投公司
 1201734370 | 故本公司
 1201734370 | 湖南郴电国际发展股份有限公司
 1201734454 | 甘肃亚盛实业
 1201734454 | 甘肃亚盛实业
 1201734455 | 黑河黄藏寺水利枢纽工程
 1201734455 | 甘肃亚盛实业
 1201734455 | 甘肃亚盛实业
 1201734455 | 甘肃亚盛实业
 1201734457 | 山丹县芋兴粉业有限责任公司
 1201734457 | 甘肃天润薯业有限责任公司
 1201734457 | 甘肃大有农业科技有限公司
 1201734457 | 甘肃亚盛薯业有限责任公司
 1201734457 | 甘肃亚盛薯业有限责任公司
 1201734457 | 甘肃亚盛薯业有限责任公司
 1201734457 | 山丹县芋兴粉业有限责任公司
 1201734457 | 山丹县芋兴粉业有限责任公司
 1201734457 | 甘肃天润薯业有限责任公司
 1201734457 | 甘肃天润薯业有限责任公司
 1201734457 | 甘肃大有农业科技有限公司
 1201734457 | 甘肃天润薯业有限责任公司
 1201734457 | 甘肃大有农业科技有限公司
 1201734457 | 芋兴粉业公司
 1201734457 | 天润薯业公司
 1201734457 | 甘肃亚盛实业
 1201734458 | 甘肃天润薯业有限责任公司
 ...
```
Again, to start, we declare the schema for our company_candidate table here just the two names, and the two person_mention IDs referred to .

```
@extraction
transaction_candidate(
    p1_id   text,
    p1_name text,
    p2_id   text,
    p2_name text
).
```

We simply construct a table of person counts, and then do a join with our filtering conditions. In DDlog this looks like

``` js
num_company(doc_id, sentence_index, COUNT(p)) :-
    company_mention(p, _, doc_id, sentence_index, _, _).


function map_transaction_candidate over (
        p1_id         text,
        p1_name       text,
        p2_id         text,
        p2_name      text
    ) returns rows like transaction_candidate
    implementation "udf/map_transaction_candidate.py" handles tsv lines.


transaction_candidate += map_transaction_candidate(p1, p1_name, p2, p2_name) :-
    num_company(same_doc, same_sentence, num_p),
    company_mention(p1, p1_name, same_doc, same_sentence, p1_begin, _),
    company_mention(p2, p2_name, same_doc, same_sentence, p2_begin, _),
    num_p < 5,
    p1_name != p2_name,
    p1_begin != p2_begin.
```

Again, to run, just compile and execute as in previous steps.

```
$ deepdive compile && deepdive do  transaction_candidate
```

```
$ deepdive sql "SELECT p1_id,p1_name, p2_id, p2_name FROM  transaction_candidate"
```

![2018-10-14-13-40-09](http://www.xdpie.com/2018-10-14-13-40-09.png)

PS：此处如果报路径错误，请将transform.py中company_full_short.csv的相对路径改为绝对路径

## Extracting features for each candidate
We will extract a set of features for each candidate:

```
spouse_feature(
    p1_id   text,
    p2_id   text,
    feature text
).
```
> Note that getting the input for this UDF requires joining the person_mention and sentences tables:

```
function extract_transaction_features over (
 p1_id text,
 p2_id text,
 p1_begin_index int,
 p1_end_index int,
 p2_begin_index int,
 p2_end_index int,
 doc_id text,
 sent_index int,
 tokens text[],
 lemmas text[],
 pos_tags text[],
 ner_tags text[],
 dep_types text[],
 dep_tokens int[]
) returns rows like transaction_feature
implementation "udf/extract_transaction_features.py" handles tsv lines.

```

```
transaction_feature += extract_transaction_features(
p1_id, p2_id, p1_begin_index, p1_end_index, p2_begin_index, p2_end_index,
doc_id, sent_index, tokens, lemmas, pos_tags, ner_tags, dep_types, dep_tokens
) :-
company_mention(p1_id, _, doc_id, sent_index, p1_begin_index, p1_end_index),
company_mention(p2_id, _, doc_id, sent_index, p2_begin_index, p2_end_index),
sentences(doc_id, sent_index, _, tokens, lemmas, pos_tags, ner_tags, _, dep_types,
dep_tokens).
```

Again, to run, just compile and execute as in previous steps.

`$ deepdive compile && deepdive do transaction_feature`

`$ deepdive sql "SELECT * FROM transaction_feature order by random() limit 20" `

```

 p1_id          |          p2_id          |            feature             
-------------------------+-------------------------+--------------------------------
 1201746717_12_351_356   | 1201746717_12_1061_1062 | NGRAM_1_[集团]
 1201746717_11_220_225   | 1201746717_11_941_946   | NGRAM_1_[40]
 1201746717_11_1223_1227 | 1201746717_11_187_188   | INV_NGRAM_3_[种植 、 销售]
 1201746717_11_515_516   | 1201746717_11_1097_1102 | NGRAM_1_[公司]
 1201746717_12_48_53     | 1201746717_12_627_631   | NGRAM_3_[王冰 物业 管理]
 1201746717_11_1115_1116 | 1201746717_11_90_95     | INV_NGRAM_3_[鹏欣 集团 之]
 1201747719_55_30_31     | 1201747719_55_8_9       | INV_W_NER_L_2_R_1_[O O]_[O]
 1201746717_10_323_328   | 1201746717_10_658_659   | NGRAM_2_[的 生产]
 1201746717_11_1046_1051 | 1201746717_11_167_172   | INV_NGRAM_3_[服务 鹏欣 集团]
 1201746717_11_796_801   | 1201746717_11_135_136   | INV_NGRAM_2_[控股 公司]
 1201746717_10_1084_1085 | 1201746717_10_280_281   | INV_NGRAM_2_[有限 公司]
 1201746717_11_961_962   | 1201746717_11_135_136   | INV_NGRAM_2_[公司 64]
 1201746717_11_988_989   | 1201746717_11_362_363   | INV_NGRAM_2_[公司 55]
 1201746717_11_573_578   | 1201746717_11_1302_1307 | NGRAM_3_[公司 69 启东]
 1201746717_11_573_578   | 1201746717_11_64_71     | INV_NGRAM_3_[鹏欣 集团 之]
 1201746717_10_1030_1035 | 1201746717_10_335_336   | INV_NGRAM_1_[，]
 1201746717_10_196_199   | 1201746717_10_1084_1085 | NGRAM_1_[2]
 1201746717_10_286_292   | 1201746717_10_373_374   | W_NER_L_2_R_2_[O O]_[O O]
 1201746717_11_1065_1066 | 1201746717_11_214_215   | INV_NGRAM_1_[控股]
 1201746717_10_768_769   | 1201746717_10_7_14      | INV_NGRAM_3_[农业 公司 董事长]
(20 rows)

```

# 6、Learning and inference: model specification
## Labeling data
Now, we need to use data from table transaction_dbdata which setup in previous step. this data get from http://www.gtarsc.com/DataMarket/Index , it is very authentic . First we'll declare a new table where we'll store the labels (referring to the transaction candidate mentions), with an integer value (True=1, False=-1) and a description (rule_id):

```
@extraction
transaction_label(
 @key
 @references(relation="has_transaction", column="p1_id", alias="has_transaction")
 p1_id text,
 @key
 @references(relation="has_transaction", column="p2_id", alias="has_transaction")
 p2_id text,
 @navigable
 label int,
 @navigable
 rule_id text
).
```

copy transaction_candidate to transaction_label .

```
 transaction_label(p1, p2, 0, NULL) :- transaction_candidate(p1, _, p2, _).
```

copy transaction_dbdata to transaction_label , these data is get from authiority so it have high priority =3. 

```
transaction_label(p1,p2, 3, "from_dbdata") :-
 transaction_candidate(p1, p1_name, p2, p2_name), transaction_dbdata(n1, n2),
 [ lower(n1) = lower(p1_name), lower(n2) = lower(p2_name) ;
 lower(n2) = lower(p1_name), lower(n1) = lower(p2_name) ].
 ```

We can also create a supervision rule which does not rely on any secondary structured dataset.

```
 function supervise over (
 p1_id text, p1_begin int, p1_end int,
 p2_id text, p2_begin int, p2_end int,
 doc_id text,
 sentence_index int,
 sentence_text text,
 tokens text[],
 lemmas text[],
 pos_tags text[],
 ner_tags text[],
 dep_types text[],
 dep_tokens int[]
) returns (
 p1_id text, p2_id text, label int, rule_id text
)
implementation "udf/supervise_transaction.py" handles tsv lines.

```

```
$ deepdive do transaction_label_resolved
```

7、 Learning and inference: model specification

Specifying prediction variables

```
@extraction
has_transaction?(
 p1_id text,
 p2_id text
).
```


```
has_transaction(p1_id, p2_id) = if l > 0 then TRUE
 else if l < 0 then FALSE
 else NULL end :- transaction_label_resolved(p1_id, p2_id, l).
```

```
$ deepdive compile && deepdive do has_transaction
```


## Specifying connections between variables

Finally, we can specify dependencies between the prediction variables, with either learned or given weights. Here, we'll specify two such rules, with fixed (given) weights that we specify. First, we define a symmetry connection, namely specifying that if the model thinks a person mention p1 and a person mention p2 indicate a spousal relationship in a sentence, then it should also think that the reverse is true, i.e., that p2 and p1 indicate one too:

```
@weight(f)
has_transaction(p1_id, p2_id) :-
 transaction_candidate(p1_id, _, p2_id, _),
 transaction_feature(p1_id, p2_id, f).
```

First, we define a symmetry connection, namely specifying that if the model thinks a company mention p1 and a company mention p2 indicate a transaction relationship in a sentence, then it should also think that the reverse is true, i.e., that p2 and p1 indicate one too

```
 @weight(3.0)
 has_transaction(p1_id, p2_id) => has_transaction(p2_id, p1_id) :-
 transaction_candidate(p1_id, _, p2_id, _).
 ```

 ```
$ deepdive compile && deepdive do probabilities
 ```

 ```
$ deepdive sql "SELECT p1_id, p2_id, expectation FROM has_transaction_label_inference
ORDER BY random() LIMIT 20"
 ```

 ```
   p1_id        |        p2_id         | expectation 
---------------------+----------------------+-------------
 1201746717_2_48_53  | 1201746717_2_12_17   |       0.014
 1201746720_22_31_37 | 1201746720_22_6_14   |       0.002
 1201739148_9_47_48  | 1201739148_9_68_74   |       0.003
 1201738764_11_27_33 | 1201738764_11_8_13   |       0.011
 1201747871_35_0_1   | 1201747871_35_35_36  |       0.001
 1201738909_39_50_52 | 1201738909_39_46_48  |       0.029
 1201746701_2_71_71  | 1201746701_2_7_12    |       0.002
 1201738909_24_77_83 | 1201738909_24_54_65  |       0.082
 1201747748_1_9_11   | 1201747748_1_0_6     |       0.138
 1201738707_2_66_71  | 1201738707_2_117_120 |       0.008
 1201738707_7_22_28  | 1201738707_7_17_20   |       0.079
 1201734457_3_46_51  | 1201734457_3_22_27   |           0
 1201743500_3_0_5    | 1201743500_3_8_14    |       0.074
 1201739153_2_35_40  | 1201739153_2_4_10    |       0.118
 1201734457_24_19_20 | 1201734457_24_28_29  |       0.023
 1201738909_23_43_45 | 1201738909_23_35_37  |       0.014
 1201739148_9_96_104 | 1201739148_9_47_48   |           0
...
 ```