＃ fff

今年QCon大会，蚂蚁金服发布了开源SQLConnectAI产品SQLFlow，旨在“降低人工智能应用的技术门槛，让技术人员调用AI像SQL一样简单”。
SQLFlow的思想最早可以追溯到2005年，当时Teradata的提出了in-database分析[1]，将数据库与数据挖掘、机器学习相结合，
扩充了SQL的能力，降低了企业应用机器学习技术的门槛。

In-databse分析具有如下特点：
1. 掌握SQL的技术人员即可完成大部分的机器学习模型训练及预测任务，TensorFlow和Scikit－learn的专家少但是SQL专家却很多
2. 减少数据的移动，存储在数据库中的数据在原地进行机器学习建模和推理
3. 可扩展性，单机机器学习到duo
4. 通用性，即支持的机器学习算法的丰富性。


In-databse分析的痛点问题

下面，我们按照时间线梳理一下In-databse分析的重要学术论文和工业界产品。
前面提到，

BigQuery ML 最早于2018年7月发布, 它在BigQeury数据仓库中内嵌了基于SQL的机器学习算法，技术人员不需要移动数据，也不需要使用Python或者R，就可以以类SQL的语法直接调用机器学习算法训练模型和预测。
下面的例子展示了如何在BigQeury ML中训练和使用线性回归模型。
'''
CREATE MODEL income_model
 OPTIONS (model_type=‘linear_reg’, labels=[‘income’])
 AS SELECT state, job, income FROM census_data;
SELECT predicted_income FROM PREDICT(MODEL ‘income_model’,
 SELECT state, job FROM customer_data);
'''




https://conf.slac.stanford.edu/xldb2019/sites/xldb2019.conf.slac.stanford.edu/files/Wed_10.55_Seyed_Umar_BigQueryML-XLDB2019.pdf

Spark SQL: Relational Data Processing in Spark
