# digitaldev-deeplens-chocolate
Code related to deploying a custom object detection project onto the AWS Deeplens

With regards to how the Amazon services tie into each other and the underlying processes involved:
-	Create an s3 bucket
-	Establish a labelling job using Amazon Ground Truth
-	Create a training job with this labelled data to create a model
-	Optimise the model for Deeplens usage
-	Create and deploy an inference Lambda function for the Deeplens to call
-	Create a project using this model and lambda function for the Deeplens to utilise
-	View Deeplens Output

Step 1: Data Collation

For image-based machine learning, it is important that the pictures of a particular object/environment are varied in lighting and backgrounds etc. otherwise the model will only be able to detect object in a very specific manner, which is not ideal for most real-world applications. It can change for what the subject is and what context you plan to use it, but it is good to have a sample size of at least 200-250 images for the purpose that we intend to use it for. 
If the images are not taken in .jpg, they should be converted as such. It is very much preferred that the images taken are in square format (1:1). If they are not, the Sagemaker training job will crop the images to a square shape automatically but at a risk that it can cut out the key object of interest. The ideal resolution for the images is either 300 by 300, or 512 or 512. However, for the training job that was used for testing, 480 x 480 performed well also. Just ensure that you get this size correct when you input your Sagemaker training parameters.

Step 2: Storage (s3)

For the labelling job, training job, model data and other functionalities, we need to create a central storage location. Navigate to the S3 console and create a new bucket. IMPORTANT: Ensure that the s3 bucket-name begins with “deeplens- “or else the Deeplens will not be able to read the data. Once the bucket is created, creating sub folders of “input” and “output” will be helpful, and what will be used for the examples and sample notebook provided. Upload the images directly into the “input” folder within the s3 bucket.

Step 3: Labelling Job (Sagemaker: Ground Truth)

-	Navigate to AWS Sagemaker and under Labelling jobs, select Create Labelling job.
-	Select Automated data setup
-	For input, select the location of the “input” folder within the s3 bucket you created
-	For output, select “Specify a new location” and select the output folder within the s3 bucket you created
-	For data Type, we want “image”
-	Click “Complete data setup”
-	Select the relevant task type, for object detection we will choose “Bounding box”, then click Next
-	For Worker type, you can choose public should you not want to label the images yourself and are prepared to pay for the service. Alternatively select private, and then select/create a labelling team which you can add workers to yourself
-	Fill out the description section of the labelling screen and the relevant tags for the objects you want to be labelled.
-	Click submit

Have fun Labelling!


Step 4: Training Job (Sagemaker)

-	Navigate to the Sagemaker Training tab and select training jobs
-	Select “Create training job”
-	Choose a name for the job. 
IMPORTANT: Ensure that the Sagemaker Training Job name begins with “deeplens-“ or else the deeplens will not be able to read the data.
-	Select a role that has the appropriate access
-	For algorithm source, select Amazon Sagemaker built-in algorithm, then from the drop-down menu select “object detection”
-	Our instance type needs to me of the p2 or p3 type as this is the instance kind that utilises a GPU. P2.xlarge should be fine for most cases, but if you like to live life in the fast-lane, x8 or x16 will complete your job lickety-split.


The hyperparameter section of the training job is contentious to say the least. For starters there are the parameters that we absolutely need to set:
-	Base_network = resnet-50
-	Use_pretrainged_model = 1
-	Num_traing_samples = No. of training samples generated by the notebook instance (80% of total dataset)
-	Image_shape = No. of pixels per side of the square image (if 480 by 480, then size = 480.)
-	Num classes = no. of different objects you are training the model to detect
-	Label width = 50
Some other parameters will depend on your data-set, but this is what worked for the dataset that was used in this example
-	Optimizer = sgd
-	Learning rate = 0.0005
-	Mini_batch_size = 16
-	Epochs = 50
-	lr_scheduler_step = 5,10,25,40 (based directly on number of epochs)
The rest of the values are the default ones provided.

For input data configuration, you will see a channel named “train”. Within that channel you will need to do the following:
-	Select input mode as “Pipe”
-	Record Wrapper as “RecordIO”
-	S3 data type as “Augmented Manifest File”
-	For the Augmented Manifest File attribute names, type “source-ref”, then add a row and type your labelling job name, e.g. “objectlabellingjob2021”
-	Input the relevant bucket path for the manifest file. This is created by utilising the notebook instance that has been provided
- You will then need to add another channel and name this one “validation”. Repeat all the above steps for this new validation channel, except for the s3 location, as the notebook instance will provide you a unique location for this channel as well.

After providing the location of your output folder with the s3 bucket for output data configuration, you can begin your training job.
After a short period, the training status will update from “starting” to “training”. Once this has occurred, we can monitor its progress. The value of interest with these training job is the validation m:AP variable. The higher this value is, the more accurate your model is performing (0 < m:AP < 1). Whilst this value does occur in a graphical form once enough epochs have been executed, a much better way of viewing the training job output is by pressing “View logs” above these graphs. Select the stream that is relevant to your training job and then ctrl + f “validation”. You will see each occurrence of the individual epochs’ validation m:AP value, and if it achieves a new highest value for this variable, it will update the model based on that epoch.

Once the Sagemaker training job is completed, this is also how you should observe the final result as well.

OPTIONAL:
If you are not getting results that are quite as high as you like, you can use a hyperparameter tuning job to refine various constants that were listed above to match your desired model. Repeat the same parameters as listed above, with two alterations:
-	Have learning rate as continuous: 0.0001 – 0.01
-	Ensure all 4 optimizers are an option for the tuning job
-	Keep pretty much all other parameters the same

The number of training jobs is up to you, but we suggest setting 30-50, with only no parallel jobs running as to make the most of the early stopping mechanisms of the tuning job.

Step 5: Create Model (Sagemaker/Desktop)

Select the training job that you created and select create a model. There are no special requirements for this model in particular. Once that is done, you can also create an endpoint for the model as well. Easy!

The following is only necessary for single-shot object detection. For some reason there are layering issues with this type of code, so we must rectify the model artifacts manually in order for it to be readable on the deeplens. 

First, navigate to the to the output of the training job in s3 where the model artifacts lie (bucketname/output/trainingjobname/output). The relevant file should be named “model.tar.gz”. Save this file to your computer, and then using a zip tool such as 7zip, extract the files that lie within that destination. There should be 3 files named “model_algo_1.json”, “model_algo_1.params” etc. You will need to rename these model artifacts’ prefixes to whatever suits the context of your project e.g. jhg_choc_model_algo_1, ensuring to keep the rest of the name intact
.
The alteration of the model artifacts must be carried out using a python script that we can grab from git.
IMPORTANT: Script must be run using python 2 and not python 3, so ensure python 2 is installed appropriately.

Input this command into gitbash: git clone https://github.com/zhreshold/mxnet-ssd.git
Now navigate to the where the mxnet folder has been saved. You will need to move the models that you have unzipped and renamed into the “Models” folder. NOW cd into the mxnet folder from the command window and install mxnet using this command: “mxnet==1.1.0.post0”. Once mxnet is installed in this folder, input the following command to rectify the model artifacts: 

python deploy.py --network vgg16_reduced --prefix model/jhg_choc_model_algo_1 --nms 0.45 --data-shape 480 --num-class 2

Obviously you will need to adapt this command to the training job that you utilised to create this model, where – prefix is the prefix you renamed your model artifacts to. If this command has been executed correctly, there will now be the same files but with two new artifacts starting with “deploy_”. Take note of this name as you will need to use it in your lambda function for inference. You will need to use 7zip to turn these files back into “model.tar.gz”. You need to do this by turning them into a “.tar” first and then subsequently a “.tar.gz”. Upload these model artifacts to the original s3 location with the same name, replacing the old one. 

Step 6: Create Inference Function (Lambda)

Navigate to the AWS Lambda console and create a function. Ensure that the Lambda function name begins with “deeplens-“ or else the deeplens will not be able to read the data. The Lambda is provided within this Git Repository, with some minor alterations needed to be made to cater for your specific project variables, which is highlighted in the code itself.

Once you have created this lambda, click save all, then “Deploy” and then scroll up to the top of the page and select “Publish new Version”. You will need to do these steps every time you update your lambda before re-uploading it onto the deeplens.

Step 7: Initiate Project and view output (Deeplens)

We can now finally upload our custom project onto the deeplens. Go into projects and create a new project, importing the correct lambda function and model. Upload this project onto the deeplens device you wish to use. 
Once it has finished uploading, there are a variety of ways we can view the output of the deeplens. The first one, which can also be utilised for troubleshooting, is using the AWS IoT output. Within your device details, you should see a url to the AWS IoT Thing that relates to the device, plus a link to the AWS IoT console. Click this link and subscribe to the URL that is given to you. If the device is not operating correctly, you should get an error message that relates to a line of code within your lambda. If it is working correctly you should get an output of “{}” with the object scores within these brackets when the deeplens sees one of the objects it has been trained to detect. 

There are two ways to view the project feed that actually draws the bounding box around these objects. The first is via the project stream on the Deeplens Console on your browser. On the same page you attained the AWS IoT URL, you should find and option to view the livestream. Once installing the relevant certificate, you should be able to see the project view, as well as the regular video view for the machine as well. Note that in order for this to work, the device must be successfully connected to the internet, which can be checked either on the device settings or the wifi-indicator on the deeplens itself. The final way to view the output is via the deeplens itself. Connect a HDMI-MicroUSB cable from the deeplens to the monitor and input the password into the device (can be found via the SSH settings in the deeplens console). Open a command window and input the following command: 

mplayer -demuxer lavf -lavfdopts format=mjpeg:probesize=32 /tmp/results.mjpeg

The feed should appear now in a new window. Note that this feed is only viable if a resnet50 framework was utilised in the training job. 



