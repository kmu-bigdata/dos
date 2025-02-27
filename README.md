# DoS

Dense or Sparse : Optimal SPMM-as-a-Service for Big-Data Processing

[Paper](https://ddps-publications.s3.ap-northeast-1.amazonaws.com/dos-tbd.pdf)
<br>
[Demo Page](http://dos.ddps.cloud/)

<br>

## How to use data-generation and DoS

### 1. Install Docker on Amazon EC2(Ubuntu18.04, t2.medium, 20GB)

```
sudo apt-get update -y
sudo apt-get remove docker docker-engine docker.io
sudo apt-get install docker.io -y
sudo service docker start
sudo chmod 666 /var/run/docker.sock
sudo usermod -a -G docker ubuntu
```

### 2. Run Container

```
sudo docker pull tensorflow/tensorflow:2.5.0
sudo docker run -it tensorflow/tensorflow:2.5.0 bash
```

### 3. Setting

```
apt-get update -y
apt-get install git -y
DEBIAN_FRONTEND=noninteractive apt-get install r-base -y
cd home
git clone https://github.com/kmu-bigdata/dos.git
cd dos
pip install -r requirements.txt
```

### 4. Data Generation

```
cd data-generation
./generate-and-optimize-lhs-data.sh
# SPMM based on /data/optimal-lhs-data.csv
./generate-trainset-testset.sh
```

### 5. Train DoS

```
cd ../dos
python3 train.py
```

### 6. Test DoS

```
python3 test.py
```

### 7. Inference DoS

```
python3 inference.py --nr_l 10000 --nc_l 60000 --nc_r 20000 --d_l 0.0001 --d_r 0.03 --nnz_l 60000 --nnz_r 36000000
```

<br><br>

## How to create Microservice using AWS

### 1. Setting on Amazon EC2(Ubuntu18.04, t2.medium, 20GB)

```
sudo apt-get update -y
sudo apt-get install git -y
sudo apt-get install awscli -y
git clone https://github.com/kmu-bigdata/dos.git
```

### 2. Install Docker

```
sudo apt-get remove docker docker-engine docker.io
sudo apt-get install docker.io -y
sudo service docker start
sudo chmod 666 /var/run/docker.sock
sudo usermod -a -G docker ubuntu
```

### 3. Build Container Image using Dockerfile

```
cd dos/microservice
docker build -t "image-name" .
```

### 4. Create Amazon ECR Repository

- Create Amazon ECR Repository to store container images.

### 5. Upload Container Image to Amazon ECR

```
aws configure

export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile

docker tag "image-name" $ACCOUNT_ID.dkr.ecr."region-name".amazonaws.com/"ecr-name"
aws ecr get-login-password --region "region-name" | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr."region-name".amazonaws.com
docker push $ACCOUNT_ID.dkr.ecr."region-name".amazonaws.com/"ecr-name"
```

### 6. Create a AWS Lambda based on Amazon ECR Container Image

- The container image in Amazon ECR is specified as the runtime environment of the AWS Lambda function.
- When configuring the Lambda, set the memory to 512 MB and the timeout to 1 minute.

### 7. Write a Lambda function that recommends an optimal multiplication method based on matrix multiplication information

- The Lambda function predicts and transmits the optimal SPMM method according to the matrix multiplication arguments using the Sparse X Sparse Latency prediction model and the Sparse X Dense Latency prediction model.
- Lambda functions can be written based on dos/microservice/lambda_function.py.

### 8. Create Amazon API Gateway and connect AWS Lambda trigger

- The AWS Lambda function receives a matrix multiplication argument from Amazon API Gateway.
- After that, the optimal SPMM method according to the matrix multiplication argument is sent back to Amazon API Gateway.

<br><br>

## How to build spark-3.1.2

### 1. Setting on Amazon EMR-6.4.0

```
sudo yum update -y
sudo yum install git -y
cd /home/hadoop
git clone https://github.com/kmu-bigdata/dos.git
```

### 2. Build

```
cd dos/spark-3.1.2 && ./build/mvn -pl :spark-mllib_2.12 -DskipTests clean install
```

### 3. Replace existing MLLib jar file in Amazon EMR with new build jar file

```
sudo mv /home/hadoop/dos/spark-3.1.2/mllib/target/spark-mllib_2.12-3.1.2.jar /usr/lib/spark/jars/spark-mllib_2.12-3.1.2-amzn-0.jar
```

### 4. Run Spark

```
spark-shell
```

### 5. Simple way to use DoS

- SparseMatrix Multiplication
    
    ```
    import org.apache.spark.mllib.linalg.SparseMatrix
    import java.util.Random;
    
    val NumRow_L = 2
    val NumCol_L = 3
    val NumCol_R = 3
    val D_L = 0.001
    val D_R = 0.005
    
    val l_sm = SparseMatrix.sprand(NumRow_L, NumCol_L, D_L, new Random(24))
    val r_sm = SparseMatrix.sprand(NumCol_L, NumCol_R, D_R, new Random(24))
    
    l_sm.multiply(r_sm)
    ```
    
- BlockMatrix Multiplication
    
    ```
    import org.apache.spark.mllib.linalg.distributed.{CoordinateMatrix, MatrixEntry}
    
    val NumRow_L = 2
    val NumCol_L = 3
    val NumCol_R = 3
    val BlockRow_L = 1
    val BlockCol_L = 1
    val BlockCol_R = 1
    
    val l_entries = sc.parallelize(Seq((0, 0, 1.0), (1, 1, 2.0), (0, 2, 3.0), (1, 2, 4.0))).map{case (i, j, v) => MatrixEntry(i, j, v)}
    val l_block_matrix = new CoordinateMatrix(l_entries, NumRow_L, NumCol_L).toBlockMatrix(BlockRow_L, BlockCol_L).cache
    
    val r_entries = sc.parallelize(Seq((1, 0, 5.0), (2, 0, 6.0), (0, 1, 7.0), (2, 1, 8.0), (1, 2, 9.0))).map{case (i, j, v) => MatrixEntry(i, j, v)}
    val r_block_matrix = new CoordinateMatrix(r_entries, NumCol_L, NumCol_R).toBlockMatrix(BlockCol_L, BlockCol_R).cache
    
    l_block_matrix.multiply(r_block_matrix).validate
    ```
