# Distributed Image Processing in Cloud Dataproc

## Task 1. Create a development machine in Compute Engine

First, you create a virtual machine to host your services.

1. In the Cloud Console, go to **Compute Engine** > **VM Instances** > **Create Instance**.

![The navigation path to the Create Instance button, which is highlighted](https://cdn.qwiklabs.com/3U1qCvvhTTw%2BqvK%2FuZxEnB2BZXmH%2BF3lPefxGzP6EK0%3D)

2. Configure the following fields and leave the others at their default value:

- **Name**: devhost
- **Series**: N1
- **Machine Type**: 2 vCPUs (n1-standard-2 instance)
- **Identity and API Access**: Allow full access to all Cloud APIs.

![The create instance page displaying the populated fields mentioned in step 2.](https://cdn.qwiklabs.com/H1G5suwtrbm43PL31cBqLyW9WpqekGZ2tKg%2F%2B6J4iNk%3D)

3. Click **Create**. This will serve as your development ‘bastion' host.
4. Now SSH into the instance by clicking the **SSH** button on the Console.

## Task 2. Install software

Now set up the software to run the job. Using `sbt`, an open source build tool, you'll build the JAR for the job you'll submit to the Cloud Dataproc cluster. This JAR will contain the program and the required packages necessary to run the job. The job will detect faces in a set of image files stored in a Cloud Storage bucket, and write out image files with the faces outlined, to either the same or to another Cloud Storage bucket.

1. Set up Scala and sbt. In the SSH window, install `Scala` and `sbt` with the following commands so that you can compile the code:

   ```
   sudo apt-get install -y dirmngr unzip
   sudo apt-get update
   sudo apt-get install -y apt-transport-https
   ```

   ```linux
   echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
   echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
   curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
   ```

   ```linux
   sudo apt-get update
   sudo apt-get install -y bc scala sbt
   ```

   

2. Run the following commands **in the SSH window**:

```
sudo apt-get update
gsutil cp gs://spls/gsp124/cloud-dataproc.zip .
unzip cloud-dataproc.zip
cd cloud-dataproc/codelabs/opencv-haarcascade
```

3. Launch build. This command builds a "fat JAR" of the Feature Detector so that it can be submitted to Cloud Dataproc:

```
sbt assembly
```



## Task 3. Create a Cloud Storage bucket and collect images

Now that you built your Feature Detector files, create a Cloud Storage bucket and add some sample images to it.

1. Fetch the Project ID to use to name your bucket:

```
GCP_PROJECT=$(gcloud config get-value core/project)
```

2. Name your bucket and set a shell variable to your bucket name. The shell variable will be used in commands to refer to your bucket:

````
MYBUCKET="${USER//google}-image-${RANDOM}"
````

```
echo MYBUCKET=${MYBUCKET}
```

3. Use the `gsutil` program, which comes with `gcloud` in the Cloud SDK, to create the bucket to hold your sample images:

```
gsutil mb gs://${MYBUCKET}
```

4. Download some sample images into your bucket:

```
curl https://www.publicdomainpictures.net/pictures/20000/velka/family-of-three-871290963799xUk.jpg | gsutil cp - gs://${MYBUCKET}/imgs/family-of-three.jpg
```

```
curl https://www.publicdomainpictures.net/pictures/10000/velka/african-woman-331287912508yqXc.jpg | gsutil cp - gs://${MYBUCKET}/imgs/african-woman.jpg
```

```
curl https://www.publicdomainpictures.net/pictures/10000/velka/296-1246658839vCW7.jpg | gsutil cp - gs://${MYBUCKET}/imgs/classroom.jpg
```

5. Run this to see the contents of your bucket:

```
gsutil ls -R gs://${MYBUCKET}
```

## Task 4. Create a Cloud Dataproc cluster

1. Run the following commands **in the SSH window** to name your cluster and to set the `MYCLUSTER` variable. You'll be using the variable in commands to refer to your cluster:

```
MYCLUSTER="${USER/_/-}-qwiklab"
echo MYCLUSTER=${MYCLUSTER}
```

2. Set a global Compute Engine region to use and create a new cluster:

```
gcloud config set dataproc/region us-west1
```

```
gcloud dataproc clusters create ${MYCLUSTER} --bucket=${MYBUCKET} --worker-machine-type=n1-standard-2 --master-machine-type=n1-standard-2 --initialization-actions=gs://spls/gsp010/install-libgtk.sh --image-version=2.0  
```

3. If prompted to use a zone instead of a region, enter **Y**.

   This might take a couple minutes. The default cluster settings, which include two worker nodes, should be sufficient for this lab. `n1-standard-2` is specified as both the worker and master machine type to reduce the overall number of cores used by the cluster.

   For the `initialization-actions` flag, you are passing a script which installs the `libgtk2.0-dev` library on each of your cluster machines. This library will be necessary to run the code.

## Task 5. Submit your job to Cloud Dataproc

In this lab the program you're running is used as a face detector, so the inputted `haar` classifier must describe a face. A `haar` classifier is an XML file that is used to describe features that the program will detect. You will download the [haar classifier file](https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml) and include its Cloud Storage path in the first argument when you submit your job to your Cloud Dataproc cluster.

1. Run the following command **in the SSH window** to load the face detection configuration file into your bucket:

```
curl https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml | gsutil cp - gs://${MYBUCKET}/haarcascade_frontalface_default.xml
```

2. Use the set of images you uploaded into the `imgs` directory in your Cloud Storage bucket as input to your Feature Detector. You must include the path to that directory as the second argument of your job-submission command.

- Submit your job to Cloud Dataproc:

```
cd ~/cloud-dataproc/codelabs/opencv-haarcascade
```

```
gcloud dataproc jobs submit spark \
--cluster ${MYCLUSTER} \
--jar target/scala-2.12/feature_detector-assembly-1.0.jar -- \
gs://${MYBUCKET}/haarcascade_frontalface_default.xml \
gs://${MYBUCKET}/imgs/ \
gs://${MYBUCKET}/out/
```

You can add any other images to use to the Cloud Storage bucket specified in the second argument.

3. Monitor the job, in the Console go to **Navigation menu** > **Dataproc** > **Jobs**.

1. When the job is complete, go to **Navigation menu** > **Cloud Storage** and find the bucket you created (it will have your username followed by `student-image` followed by a random number) and click on it.
2. Click on an image in the **Out** directory.
3. Click on **Download** icon, the image will download to your computer.

How accurate is the face detection? The [Vision API ](https://cloud.google.com/vision/)is a better way to do this, since this sort of hand-coded rules don't work all that well. You can see how it works next.

1. (Optional) In your bucket go to the `imgs` folder and click on the other images you uploaded to your bucket. This will download the three sample images. Save them to your computer.
2. Click on this link to go to the [Vision API](https://cloud.google.com/vision/) page, scroll down to the **Try the API** section and upload the images you downloaded from your bucket. You'll see the results of the image detection in seconds. The underlying machine learning models keep improving, so your results may not be the same:

![Face detection on the woman](https://cdn.qwiklabs.com/Y0PtrYuy%2Fb%2FpnwqKuCbkVuz8DIl6RE7ISSnBIgxXTqg%3D) ![Face detection in the classrom](https://cdn.qwiklabs.com/vMs2El4Bsp%2BNK7B1rZAPEaCOBv1YBkUSc0P3PX%2FAyTM%3D) ![Face detection on the family of three](https://cdn.qwiklabs.com/HJUW8VYV70K%2B3dfRHfnB4chtWxSLcS2trpwHrsl%2BncM%3D)