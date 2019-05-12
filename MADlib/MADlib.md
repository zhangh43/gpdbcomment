# SQL与机器学习

今年QCon大会，蚂蚁金服发布了开源SQLConnectAI产品SQLFlow，旨在“降低人工智能应用的技术门槛，让技术人员调用AI像SQL一样简单”。
SQLFlow的思想最早可以追溯到2005年，当时Thomas Tileston提出了in-database分析，将数据库与数据挖掘、机器学习有机地统一了起来。
In-database分析通过扩充SQL的能力，降低了企业应用机器学习技术的门槛，同时解决了数据在不同系统间移动所产生的一系列问题。

In-databse分析主要用于解决以下问题：
1. 降低机器学习门槛，掌握SQL的技术人员即可完成大部分的机器学习模型训练及预测任务，TensorFlow和Scikit－learn的专家少但是SQL专家却很多
2. 减少数据的移动，存储在数据库中的数据在原地进行机器学习建模和推理
3. 可扩展性，单机机器学习到集群机器学习的扩展
4. 通用性，即支持的机器学习算法的丰富性。

从架构来区分，In-database分析可以分为四种：
1. SQL wrapper, 将扩展的SQL语法翻译成AI引擎可以执行的程序，它不涉及对机器学习算法的重构，而是直接调用底层AI引擎。
2. SQL on AI engine, 让AI引擎支持SQL语法及相应的查询优化器，但SQL的执行器只用AI引擎自身，从而实现了AI engine as a database的架构。
3. SQL based AI algotithm, 基于SQL重写机器学习算法，并将这些算法内置到数据库中。
4. UDA based AI algorithm, 基于数据库的User Defined Agrregation（UDA）实现机器学习算法。基于不同场景，UDA语言可以是Python，R或者是高效的C／C++。这些算法同样被内置到数据库中。

下面我们分别从上述四种In-database分析的架构中选取典型代表，分析一下各自的适用场景。

1. SQL wrapper架构的代表是前面提到的SQLFlow，其偏重于解决In-databse分析中的问题1和问题4。问题1，即降低机器学习门槛，SQLFlow通过为SQL扩充TRAIN和PREDICT语法，实现了技术人员直接调用SQL即可实现模型训练和预测。
```
sqlflow> SELECT *
FROM iris.train
TRAIN DNNClassifier
WITH n_classes = 3, hidden_units = [10, 20]
COLUMN sepal_length, sepal_width, petal_length, petal_width
LABEL class
INTO sqlflow_models.my_dnn_model;
```
问题4通用性，SQLFlow支持的机器学习算法数量由底层AI引擎决定，目前其支持TensorFlow。但问题3可扩展性，SQLFlow需要将wrapper的数据库和AI引擎相结合，对于一个M结点的分布式数据库如何与N个结点的AI引擎高效结合，并不是简单wrapper可以完成的，往往需要专业的连接器，比如连接Spark和Greenplum DB的greenplum-spark-connector。对于问题2减少数据移动，SQLFlow无能为力了。

2. SQL on AI engine的代表是Spark SQL。Spark SQL于2014年5月随着SPARK1.0.0正式推出，其前身是Spark Shark。Spark SQL偏重于让Spark这个AI引擎具有处理SQL的能力，降低Spark本身的入门门槛，从这点上我们再次看到了SQL的普遍性和重要性。其偏重解决In-databse分析中的后三个问题，但它需要和Spark的DataFrame接口统一使用才能更好的发挥AI的威力，它降低了R语言使用者上手Spark的难度，但对于纯SQL技术人员，适用JDBC连接Spark SQL只能执行标准SQL查询而非带有AI能力的查询。

3. SQL based AI algotithm的代表是BigQuery ML。BigQuery ML于2018年7月发布, 它在BigQeury数据仓库中内嵌了基于SQL的机器学习算法，
技术人员不需要移动数据，也不需要使用Python或者R，就可以使用类SQL的语法直接调用机器学习算法训练模型和预测。
下面的例子展示了如何在BigQeury ML中训练和使用线性回归模型。
```
CREATE MODEL income_model
 OPTIONS (model_type=‘linear_reg’, labels=[‘income’])
 AS SELECT state, job, income FROM census_data;
SELECT predicted_income FROM PREDICT(MODEL ‘income_model’,
 SELECT state, job FROM customer_data);
```
BigQuery ML很好的解决了前三个问题，但使用SQL编写复杂的机器学习算法虽然并非不可能，但开发效率也相对较低，截至目前，BigQuery ML只支持Linear regression，Logistic regression 和K-means clustering三类机器学习算法。


4. UDA based AI algorithm的代表是Apache MADlib。MADlib于2011年诞生,是由Pivotal Greenplum DB团队和高校联合研发的，参与的大学包括伯克利大学加州分校、斯坦福大学、威斯康辛麦迪逊大学、佛罗里达大学。2017年MADlib正式毕业成为Apache顶级项目。MADlib基于数据库User Defined Agrregation（UDA）实现机器学习算法，它完美的解决了In-databse分析的四个问题。

MADlib通过将机器学习算法封装成数据库的UDF，方便SQL用户，其用户接口如下：
```
```

Greenplum DB是世界领先的开源MPP数据库，Greenplum与MADlib结合可以实现在DB的Segment上并行执行聚集，生成sub state，并在Master结点进行state的聚合，从而实现单结点到集群的扩展。

