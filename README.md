## Pytorch on Angel 

A light-weight project which runs pytorch on [angel](https://github.com/Angel-ML/angel), providing pytorch the ability to run with high-dimensional models.

### Architecture

----

![][1]

Pytorch on Angel's architecture design consists of three modules:

  - **python client**: python client is used to generate the pytorch script module.
  - **angel ps**: provides a common Parameter Server (PS) service, responsible for distributed model storage, communication synchronization and coordination of computing.
  - **spark executor**: the worker process is responsible for data processing、load pytorch script module and communicate with the `Angel PS Server`to complete model training and prediction, especially pytorch c++ backend runs in native mode for actual computing backend.

### Compilation & Deployment Instructions by Docker

To use Pytorch on Angel, we need three components: .jar file with a set of shared libraries for pytorch c++ backend compiled by this repo, and the pytorch script module.

#### Compile .jar and the shared c++ libraries

```bash
# Below script will build the jar files and bunlde the shared c++ libraries in containers
# The generated files *.jar and angel_libtorch.zip are in ./dist
./build.sh
```

#### Generate a pytorch script model

```bash
# We have implemented some algorithms in the python/recommendation under the root directory
# Below script will generate a deepfm model deepfm.pt in ./dist
./gen_pt_model.sh python/recommendation/deepfm.py --input_dim 148 --n_fields 13 --embedding_dim 10 --fc_dims 10 5 1
```

### Compilation & Deployment Instructions Manually

#### Install Pytorch

  - pytorch >=v1.1.0 
 
we recommend using [anaconda](https://www.anaconda.com/) to install pytorch, run command:
```$xslt
conda install -c pytorch pytorch
```
pytorch detailed installation documentation can refer to [pytorch installation](https://github.com/pytorch/pytorch#installation)


#### Compiling java submodule
1. **Compiling Environment Dependencies**
   - Jdk >= 1.8
   - Maven >= 3.0.5

2. **Source Code Download**
   ```$xslt
   git clone https://github.com/Angel-ML/PyTorch-On-Angel.git
   ```

3. **Compile**  
   Run the following command in the java root directory of the source code:
   ```$xslt
   mvn clean package -Dmaven.test.skip=true
   ```
   After compiling, a jar package named 'pytorch-on-angel-&lt;version&gt;.jar' will be generated in `target` under the java root directory.


#### Compiling cpp submodule
1. **Compiling Environment Dependencies**
   - gcc >= 5
   - cmake >= 3.12

2. **LibTorch Download**
   - Download the `libtorch` package from [here](https://pytorch.org/) and extract it to the user-specified directory
   - set TORCH_HOME(path to libtorch) in `CMakeLists.txt` under the  cpp root directory
  
3. **Compile**
   Run the following command in the `cmake-build-debug` directory under the  cpp root directory:
   ```$xslt
   cmake ..
   make
   ```
   After compiling, a shared library named 'libtorch_angel.so' will be generated in `cmake-build-debug` under the  cpp root directory.
   
#### Spark on Angel deployment
pytorch on angel runs on spark on angel, so you must deploy the spark on angel client first. The specific deployment process can refer to [documentation](https://github.com/Angel-ML/angel/blob/master/docs/tutorials/spark_on_angel_quick_start_en.md).

### Quick Start
Use `$SPARK_HOME/bin/spark-submit` to submit the application to cluster in the pytorch on angel client.   
Here are the submit example for deepfm.
1. **Generate pytorch script model**  
   users can implement their own algorithms using pytorch. We have implemented some algorithms in the python/recommendation under the root directory, you can run the following command to generate a deepfm model:
   ```$xslt
   python deepfm.py --input_dim 148 --n_fields 13 --embedding_dim 10 --fc_dims 10 5 1
   ```
   After executing this command, you will get a model file named deepfm.pt

2. **Package c++ library files**
   Package the `libtorch/lib`library file with the shared library file `libtorch_angel.so` generated by the compiled cpp submodule, for example, we packaged and named it `angel_libtorch.zip`

3. **Upload training data to hdfs**
   upload training data python/recommendation/census_148d_train.libsvm.tmp to hdfs directory

4. **Submit to Cluster**  
   ```$xslt
   source ./spark-on-angel-env.sh  
   $SPARK_HOME/bin/spark-submit \
          --master yarn-cluster\
          --conf spark.ps.instances=5 \
          --conf spark.ps.cores=1 \
          --conf spark.ps.jars=$SONA_ANGEL_JARS \
          --conf spark.ps.memory=5g \
          --conf spark.ps.log.level=INFO \
          --conf spark.driver.extraJavaOptions=-Djava.library.path=$JAVA_LIBRARY_PATH:.:./torch/angel_libtorch \
          --conf spark.executor.extraJavaOptions=-Djava.library.path=$JAVA_LIBRARY_PATH:.:./torch/angel_libtorch \
          --conf spark.executor.extraLibraryPath=./torch/angel_libtorch \
          --conf spark.driver.extraLibraryPath=./torch/angel_libtorch \
          --conf spark.executorEnv.OMP_NUM_THREADS=2 \
          --conf spark.executorEnv.MKL_NUM_THREADS=2 \
          --queue $queue \
          --name "deepfm for torch on angel" \
          --jars $SONA_SPARK_JARS  \
          --archives angel_libtorch.zip#torch\  #path to c++ library files
          --files deepfm.pt \   #path to pytorch script model
          --driver-memory 5g \
          --num-executors 5 \
          --executor-cores 1 \
          --executor-memory 5g \
          --class com.tencent.angel.pytorch.examples.ClusterExample \
          ./pytorch-on-angel-1.0-SNAPSHOT.jar \   # jar from Compiling java submodule
          input:$input batchSize:128 torchModelPath:deepfm.pt \
          stepSize:0.001 numEpoch:10 partitionNum:5 \
          modulePath:$output \
   ```

### Algorithms
Currently, Pytorch on Angel supports a series of recommendation and deep graph convolution network algorithms.

1. [Recommendation Algorithms](./docs/recommendation.md)
2. [Graph Algorithms](./docs/graph.md)


[1]: ./docs/img/pytorch_on_angel_framework.png
