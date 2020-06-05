# Activity 05: Model Implementation with Predict

In this activity, you will work with your team to train a simple model and prepare it for use in Azure Synapse Analytics T-SQL queries. 

# Team Challenge 

WWI wants you to show them how automated machine learning might help them accelerate the model training effort.

1. Open the Develop hub.
2. Select Notebooks and open `Activity 05 - Model training.ipynb`.
3. Follow the instructions within the notebook.

# Activity 05: Solution

```python
df = spark.read.load('abfss://wwi-02@asadatalake177728.dfs.core.windows.net/sale-csv/sale-csv/wwi-factsale.csv', format="csv"
, header=True, sep="|"
)
```
```python
df.createOrReplaceTempView("facts")
```

```python
display(spark.sql("SELECT * FROM facts WHERE `Customer Key` == '11' ORDER BY `Stock Item Key`"))
```


```python
from pyspark.sql.functions import col
df3 = spark.sql("SELECT double(`Customer Key`) as customerkey, double(`Stock Item Key`) as stockitemkey, double(`Quantity`) as quantity FROM facts").where(col("quantity").isNotNull())
df3.cache()
```

```python
from pyspark.ml.feature import VectorAssembler

vectorAssembler = VectorAssembler(inputCols = ['customerkey', 'stockitemkey'], outputCol = 'features')
df4 = vectorAssembler.transform(df3)
df5 = df4.select(['features', 'quantity'])
df5.show(10)
```

```python
trainingFraction = 0.7
testingFraction = (1-trainingFraction)
seed = 42
```

Split the dataframe into test and training dataframes

```python
df_train, df_test = df5.randomSplit([trainingFraction, testingFraction], seed=seed)
```

```python
from pyspark.ml.regression import LinearRegression

lin_reg = LinearRegression(featuresCol = 'features', labelCol='quantity', maxIter = 10, regParam=0.3)
lin_reg_model = lin_reg.fit(df_train)

print("Coefficients: " + str(lin_reg_model.coefficients))
print("Intercept: " + str(lin_reg_model.intercept))
```


```python
df_pred = lin_reg_model.transform(df_test)
display(df_pred)
```

```python
from onnxmltools import convert_sparkml
from onnxmltools.convert.common.data_types import FloatTensorType

initial_types = [ 
    ("features", FloatTensorType([1, lin_reg_model.numFeatures])),
    # (repeat for the required inputs)
]
```


```python
model_onnx = convert_sparkml(lin_reg_model, 'sparkml GeneralizedLinearRegression', initial_types)
model_onnx
```

```python
with open("model.onnx", "wb") as f:
    f.write(model_onnx.SerializeToString())
```

ALTERNATIVE FOR BLOB STORAGE WRITING:

```python
from azure.storage.blob import BlockBlobService
 
block_blob_service = BlockBlobService(
 account_name='asadatalake177728', account_key='<data_lake_account_key>') 
 
block_blob_service.create_blob_from_text('wwi-02', '/ml/onnx/model.onnx', model_onnx.SerializeToString())
```


