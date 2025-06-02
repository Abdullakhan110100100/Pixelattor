# Pixelattor

I have created a simple event-driven image processing pipeline. The pipeline uses two S3 buckets, a source bucket and a processed bucket. When images are added to the source bucket a lambda function is triggered based on the PUT.  When invoked the lambda function receives the `event` and extracts the bucket and object information. Once those details are known, the lambda function, using the `PIL` module pixelates the image with `5` different variations (8x8, 16x16, 32x32, 48x48 and 64x64) and uploads them to the processed bucket.

# Stage 1 - Create the S3 Buckets

Move to the S3 Console
We will be creating `2` buckets, `my-source-bucket-abd-123`   and `my-dest-bucket-abd-123` respectively , all settings apart from region and bucket name is left as default.  
Click `Create Bucket` and create a bucket in the format of source-bucket-name in the `us-east-1` region  
Click `Create Bucket` and create a another bucket in the format of destination-bucket-name also in the `us-east-1` region  

# Stage 2 - Create the Lambda Role

Move to the IAM Console 
Click Roles, then Create Role  
For `Trusted entity type`, pick `AWS service`  
For the service to trust pick `Lambda`  then click `Next` , `Next` again  
For `Role name` put `PixelatorRole`  then Create the role  

Click `PixelatorRole`  
Under `Permissions Policy` we need to add permissions and it will be an `inline policy`  
Click `JSON`  and delete the contents of the code box entirely.  
Load this link in a new tab (https://github.com/Abdullakhan110100100/Pixelattor/blob/main/lambda_policy.json) 
Copy the entire contents into your clipboard and paste into the previous permissions policy code editor box  
Locate the words `REPLACEME` there should be `4` occurrences, 2 each for the source and processed buckets .. and for each of those one for the bucket and another for the objects in that bucket.  
Replace the term `REPLACEME` with the name you picked for your buckets above.
You should end with 4 lines looking like this, only with the bucket names  

```
"Resource":[
	"arn:aws:s3:::my-dest-bucket-abd-123",
	"arn:aws:s3:::my-dest-bucket-abd-123/*",
	"arn:aws:s3:::my-source-bucket-abd-123/*",
	"arn:aws:s3:::my-source-bucket-abd-123"
]
```

Locate the two occurrences of `YOURACCOUNTID`, you need to replace both of these words with your AWS account ID  
To get that, click the account dropdown at the top right   
click the small icon to copy down the `Account ID` and replace the `YOURACCOUNTID` in the policy code editor. 

```
{
	  "Effect": "Allow",
	  "Action": "logs:CreateLogGroup",
	  "Resource": "arn:aws:logs:us-east-1:123456789000:*"
  },
  {
	  "Effect": "Allow",
	  "Action": [
		  "logs:CreateLogStream",
		  "logs:PutLogEvents"
	  ],
	  "Resource": [
		  "arn:aws:logs:us-east-1:123456789000:log-group:/aws/lambda/pixelator:*"
	  ]
  }
```

Click `Review Policy`  
For name put `pixelator_access_inline`  and create the policy.  

# Stage 3 Creating a Lambda zip file

From the CLI/Terminal
Create a folder my_lambda_deployment  
Move into that folder
create a folder called lambda  
Move into that folder
Create a file called `lambda_function.py` and paste in the code for the lambda `pixelator` function (https://github.com/Abdullakhan110100100/Pixelattor/blob/main/lambda_function.py) then save 
Download this file (https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Pillow-9.0.1-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl) into that folder 
run `unzip Pillow-9.0.1-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl` and then `rm Pillow-9.0.1-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl`  
These are the Pillow module files ... required for image manipulation in Python 3.9 (which is what the lambda function will be using)  
From the same folder, run `zip -r ../my-deployment-package.zip .` which will create a lambda function zip, containing all these files in the parent directory.  


# Stage 4 - Create the Lambda Function

Move to the lambda console 
Click `Create Function`  
We're going to be `Authoring from Scratch`  
For `Function name` enter `pixelator`  
for `Runtime` select `Python 3.9`  
For `Architecture` select `x86_64`  
For `Permissions` expand `Change default execution role` pick `Use an existing role` and in the `Existing role` dropdown, pick `PixelatorRole`  
Then `Create Function`  
Click `Upload from` and select `.zip file`
locate the .zip you created yourself.
On the lambda screen, click `Upload` locate and select that .zip, and then click the `Save` button  



# Stage 5 - Configure the Lambda Function & Trigger

Click `Configuration` tab and then `Environment variables`  
We need to add an environment variable telling the pixelator function which processed bucket to use, it will know the source bucket because it's told about that in the event data.  
Click `Edit` then `Add environment variable`, under Key put `processed_bucket` and for `Value` put the bucket name of *your* processed bucket. 
Be `really really really` sure you put your `processed` bucket here and *NOT* your source bucket.  
*if you use the source bucket here, the output images will be stored in the source bucket, this will cause the lambda function to run over and over again ... bad*
*be super-sure to put your processed bucket*
Click `Save`  

Click `General configuration` then click `Edit` and change the timeout to `1` minutes and `0` seconds, then click `Save`  

Click `Add trigger`  
In the dropdown pick `S3`  
Under `Bucket` pick your *source* bucket ... *AGAIN* be really really sure this is your source bucket and *NOT* your destination bucket and *NOT* any other bucket. Only pick your *SOURCE* bucket here.  
You will need to check the `Recursive invocation` acknowledgment box, this is because this lambda function is invoked every time anything is added to the *source* bucket, if you configure this wrongly, or configure the environment variable above wrongly ... it will run the lambda function over and over again *for ever*. 
Once checked, click `Add`  

# Stage 6 - Test and Monitor

open a tab to the `cloudwatch logs` console 
make sure you have two tabs open to the `s3 console` 
In one tab open your `-source` bucket & in the other open the `-dest` bucket  

In the `-source` bucket tab, make sure to select the `Objects` tab and click `Upload`  
Add some files and click `Upload`  
Once finished, click `Close`  
Move to the `CloudWatch Logs` tab  
Click the `Refresh` icon, locate and click `/aws/lambda/pixelator`  
If there is a log stream in there, click the most recent one, if not, keep clicking the `Refresh` icon and then click the most recent log stream  
Expand the line which begins with `{'Records': [{'eventVersion':` and you can see all of the event information about the lambda invocation, you should see the object name listed in `'object': {'key'` ...
Go to the S3 Console tab for the `-processed` bucket  
Click the `Refresh` icon  
Select each of the pixelated versions of the image ... you should have 5 (`8x8`, `16x16`, `32x32`, `48x48` and `64x64`)  
Click `Open`  
You browser will either open or save all of the images  
Open them one by one, starting with `8x8` and finally `64x64` in order ... notice how they are the same image, but less and less pixelated :)  


# Conclusion
We created an event driven pixelator that will run everytime an image (not sure what will happen if any other object is uploaded since I only tested images) is uploaded to the source bucket and then a lambda function is triggered that will run and generate the different images for the different pixels and upload them to the destination bucket. Here is the original image I uploaded ![original image]((https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Images/Images_original_processed/original.jpg)) and here are the images uploaded into the destination bucket (Various pixelated images 8X8 https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Images/Images_original_processed/pixelated-8x8-aotm.jpg,  16X16 https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Images/Images_original_processed/pixelated-16x16-aotm.jpg, 32X32 https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Images/Images_original_processed/pixelated-32x32-aotm.jpg, 48X48 https://github.com/Abdullakhan110100100/Pixelattor/blob/main/Images/Images_original_processed/pixelated-48x48-aotm.jpg ) 


