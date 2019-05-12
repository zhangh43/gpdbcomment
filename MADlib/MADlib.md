# SQL与机器学习

今年QCon大会，蚂蚁金服发布了开源SQLConnectAI产品SQLFlow，旨在“降低人工智能应用的技术门槛，让技术人员调用AI像SQL一样简单”。
SQLFlow的思想最早可以追溯到2005年，当时Teradata的提出了in-database分析[1]，将数据库与数据挖掘、机器学习相结合，
通过扩充SQL的能力，降低了企业应用机器学习技术的门槛。

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

1. SQL wrapper架构的代表是前面提到的SQLFlow，其偏重于解决In-databse分析中的问题1和问题4。问题1，即降低机器学习门槛，SQLFlow

2. Spark SQL


In-databse分析的痛点问题

下面，我们按照时间线梳理一下In-databse分析的重要学术论文和工业界产品。



BigQuery ML于2018年7月发布, 它在BigQeury数据仓库中内嵌了基于SQL的机器学习算法，技术人员不需要移动数据，也不需要使用Python或者R，就可以以类SQL的语法直接调用机器学习算法训练模型和预测。
下面的例子展示了如何在BigQeury ML中训练和使用线性回归模型。

```
CREATE MODEL income_model
 OPTIONS (model_type=‘linear_reg’, labels=[‘income’])
 AS SELECT state, job, income FROM census_data;
SELECT predicted_income FROM PREDICT(MODEL ‘income_model’,
 SELECT state, job FROM customer_data);
```




https://conf.slac.stanford.edu/xldb2019/sites/xldb2019.conf.slac.stanford.edu/files/Wed_10.55_Seyed_Umar_BigQueryML-XLDB2019.pdf

Spark SQL: Relational Data Processing in Spark
