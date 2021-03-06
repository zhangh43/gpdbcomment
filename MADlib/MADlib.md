# SQL与机器学习

今年QCon大会，蚂蚁金服发布了开源SQLConnectAI产品SQLFlow，旨在“降低人工智能应用的技术门槛，让技术人员调用AI像SQL一样简单”。
SQLFlow的思想最早可以追溯到2005年，当时Thomas Tileston提出了In-database分析，将数据库与数据挖掘、机器学习有机地统一了起来。
In-database分析通过扩充SQL的能力，降低了企业应用机器学习技术的门槛，同时解决了数据在不同系统间移动所产生的一系列问题。

In-databse分析主要具有以下特性：
1. 易用性，降低机器学习门槛，掌握SQL的技术人员即可完成大部分的机器学习模型训练及预测任务，掌握TensorFlow和Scikit-learn的技术人员比掌握SQL的技术人员少很多。
2. 本地性，减少数据的移动，存储在数据库中的数据在原地进行机器学习建模和推理，提高了分析效率同时，避免了数据移动过程中存在的安全问题，减少了team间沟通成本，以及建造单独数据分析基础设施的IT成本。
3. 可扩展性，单机机器学习到集群机器学习的扩展
4. 通用性，即支持的机器学习算法的丰富性。

从2005年Thomas Tileston提出了In-database分析至今，已经涌现出很多In-database分析的产品，它们部分或全部支持In-database分析的特性，我们将主要的产品和时间线总结在图1.

从时间线可以看出，2009年MAD Skills在VLDB的发表和2011年MADlib项目的诞生可以作为In-database分析的里程碑。MADlib是由Pivotal Greenplum DB团队和高校联合研发的，参与的大学包括伯克利大学加州分校、斯坦福大学、威斯康辛麦迪逊大学、佛罗里达大学。2017年MADlib正式毕业成为Apache顶级项目。MADlib的第一篇论文“MAD skills: new analysis practices for big data”，目前Google引用已达555次，Spark SQL，BigQuery ML等产品均引用了MADlib的工作，MADlib被认为是In-database分析的先驱者和领路人。

In-database分析产品众多，不同厂商对In-database分析的理解也不尽相同。从架构来区分，In-database分析可以分为以下四种：
1. SQL wrapper, 将扩展的SQL语法翻译成AI引擎可以执行的程序，它不涉及对机器学习算法的重构，而是直接调用底层AI引擎。
2. SQL on AI engine, 让AI引擎支持SQL语法及相应的查询优化器，但SQL的执行器使用AI引擎自身，从而实现了AI engine as a database的架构。
3. SQL based AI algotithm, 基于SQL重写机器学习算法，并将这些算法内置到数据库中。
4. UDA based AI algorithm, 基于数据库的User Defined Agrregation（UDA）实现机器学习算法。基于不同场景，UDA语言可以是Python，R或者是高效的C／C++。这些算法同样被内置到数据库中。

上述四种In-database分析的架构对In-database分析的特性有不同程度的支持，我们选取SQLFlow，Spark SQL，Bigquery ML和MADlib作为这四种架构的典型代表。表1展示了四个典型In-database分析产品和它们对In-database分析特性的支持。


下面，我们结合表1，详细介绍一下SQLFlow，Spark SQL，Bigquery ML和MADlib的特点和使用场景。

1. SQL wrapper架构的代表是前面提到的SQLFlow，其偏重于解决In-databse分析中的易用性和通用性。易用性方面，SQLFlow通过为SQL扩充TRAIN和PREDICT语法，实现了技术人员直接调用SQL即可实现模型训练和预测。
```
sqlflow> SELECT *
FROM iris.train
TRAIN DNNClassifier
WITH n_classes = 3, hidden_units = [10, 20]
COLUMN sepal_length, sepal_width, petal_length, petal_width
LABEL class
INTO sqlflow_models.my_dnn_model;
```
通用性方面，SQLFlow支持的机器学习算法数量由底层AI引擎决定，目前其支持TensorFlow。但可扩展性，SQLFlow需要将wrapper的数据库和AI引擎相结合，对于M个结点的分布式数据库如何与N个结点的AI引擎高效结合的问题，并不是简单wrapper可以完成的，往往需要专业的连接器，比如连接Spark和Greenplum DB的greenplum-spark-connector。对于本地性，SQLFlow无能为力了。

2. SQL on AI engine的代表是Spark SQL。Spark SQL于2014年5月随着SPARK1.0.0正式推出，其前身是Spark Shark。Spark SQL偏重于让Spark这个AI引擎具有处理SQL的能力，降低Spark本身的入门门槛，从这点上我们再次看到了SQL的普遍性和重要性。其偏重解决In-databse分析中的后三个特性，但它需要和Spark的DataFrame接口统一使用才能更好的发挥AI的威力，它降低了R语言使用者上手Spark的难度，但对于纯SQL技术人员，使用JDBC连接Spark SQL只能执行标准SQL查询而非带有AI能力的查询（Spark SQL的UDF需要通过Spark Register不是SQL接口）。Spark SQL接口如下所示。
```
results = spark.sql(
  "SELECT * FROM people")
names = results.map(lambda p: p.name)
```

3. SQL based AI algotithm的代表是BigQuery ML。BigQuery ML于2018年7月发布, 它在Google BigQeury数据仓库中内嵌了基于SQL的机器学习算法，
技术人员不需要移动数据，也不需要使用Python或者R，就可以使用类SQL的语法直接调用机器学习算法训练模型和预测。
下面的例子展示了如何在BigQeury ML中训练和使用线性回归模型。
```
CREATE MODEL income_model
 OPTIONS (model_type=‘linear_reg’, labels=[‘income’])
 AS SELECT state, job, income FROM census_data;
SELECT predicted_income FROM PREDICT(MODEL ‘income_model’,
 SELECT state, job FROM customer_data);
```
BigQuery ML很好的解决了In-database分析的前三个特性，但使用SQL编写复杂的机器学习算法虽然并非不可能，但开发效率也相对较低，截至目前，BigQuery ML只支持Linear regression，Logistic regression 和K-means clustering三类机器学习算法。


4. UDA based AI algorithm的代表是Apache MADlib。MADlib基于数据库User Defined Agrregation（UDA）实现机器学习算法，它完美的解决了In-databse分析的四个特性。
易用性，MADlib通过将机器学习算法封装成数据库的UDF，用户可以使用标准SQL实现机器学习建模和推理，无需引入额外SQL语法，其用户接口如下：
```
SELECT madlib.logregr_train
( 'patients',                             -- Source table
  'patients_logregr',                     -- Output table
  'second_attack',                        -- Dependent variable
  'ARRAY[1, treatment, trait_anxiety]',   -- Feature vector
  NULL,                                   -- Grouping
  20,                                     -- Max iterations
  'irls'                                  -- Optimizer to use
);

SELECT p.id, madlib.logregr_predict(coef, ARRAY[1, treatment, trait_anxiety]),
       p.second_attack::BOOLEAN
FROM patients p, patients_logregr m
ORDER BY p.id;
```

本地性，MADlib的机器学习算法直接在DB的内核中执行。

可扩展性，Greenplum DB是世界领先的开源MPP数据库，Greenplum与MADlib结合可以实现在数据库的大量Segment结点上并行地执行聚集，生成sub state，并在Master结点进行sub state的聚合，从而实现机器学习算法从单结点到集群的扩展。

通用性，MADlib目前支持50多种机器学习算法。除了核心的机器学习建模和推理，MADlib还支持了数据分析流水线的全部流程，实现了数据分析的闭环。数据科学家在定义好数据分析的问题后，首先进行数据探索，分析和识别数据中可供挖掘的模式，接下来进行数据的预处理、清洗和整合，之后才进行各种类型的建模，包括非监督的数据挖掘任务、有监督的预测建模、文本分析等等。最后还要对建模后的结果进行模型选择，不同的模型有不同的评测标准，往往还会使用交叉验证等技术。MADlib对以上环节都有支持。

但包括MADlib在内的UDA based AI algorithm，需要假设模型能装入内存，因为它们将模型表达成UDA的state，对于拥有数百万、千万特征的模型，UDA based AI algorithm并不适合。

综上，笔者从SQLFlow出发，分析了In-database分析领域不同的实现架构和代表产品，它们各自有适合的应用场景，但总体上都是奔着一个目标在努力：以最高效的方式将数据和机器学习连结在一起，让更多的技术人员可以使用AI，让AI可以赋能更多的企业。更可喜的是，我们看到从SQLFlow到Spark SQL再到Apache MADlib，它们都是开源软件的一份子，开源进一步促进了这些技术的普及和深入，让AI离我们再近一些。

