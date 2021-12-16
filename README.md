# awsserverless-videosubtiltles-localization
## Table of contents
* [General info](#general-info)
* [Services Used](#services)
* [Solution Explanation](#solution)
* [Enhancements](#enhancements)
* [Implementation Steps](#implementation)

## General info
Created a small implementation to generate the captions for a video file, convert those captions into different languages and once again burn-in them with the actual video in a serverless way using AWS services. Through this solution you can generate videos with different language subtitles. Consider it as a POC as lot many features can be added both for enhancing the functionality as well as smoothen the overall work flow.

![Alt Image text](/Images/Serverlessvideocaptions.drawio.png?raw=true "Overall architecture")

## Services Used
1. **Cloudformation:** Infrastructure as a code. Whole infrastructure will be provisioned using cloudformation templates. I have tried to avoid the manual intervention as much as possible.
2. **AWS Transcribe:** AWS service which is used to extract text from the speech. This is used in a solution to get the initial subtitles from the video file. I have used MP4 for this implementation. 
3. **AWS Translate:** AWS service to translate the text into different languages. In this implementation I am converting subtitles from English to Hindi.
**AWS MediaConvert:** AWS services which supports a broad range of functionalities to easily create VOD (Video on Demand). In this solution it is used to burn-in the captions in the existing video files.
4. **AWS S3 bucket:** S3 is a storage which is holding various input and output files for different services.
5. **AWS IAM:** Roles and policies for each of the services are configured through AWS IAM. 
6. **AWS Lambda:** It is a serverless piece of AWS. In this solution it is used to trigger different actions and the services.
7. **AWS Cloudwatch Logs:** To capture the logs of each of the services used in this solution.

## Solution Explanation
Three buckets will be created one for each of the services used.
1. 	**Transcribe bucket:** It will have two folders “transinput” and “transoutput”. Original video file will be uploaded to the “transinput” folder of the bucket.
      * Input folder will have event notification which will trigger lambda function which has a transcribe job template and it will execute the AWS transcribe service.
      * Transcribe will generate the captions in the WEBVTT or SRT format, for this tutorial we have considered WEBVTT. You can open this file using notepad or sublime etc.
      * Captions file will be placed in the “transoutput” folder of the bucket.
      * Event notification will be triggered as soon as the file will be generated in the “transoutput” folder. Lambda function will be called which will copy WEBVTT caption file to the translate bucket.
2.	**Translate bucket:** Caption file will be copied to the “input” folder of the Translate bucket.
    * As soon as the file is uploaded to the input folder, it will trigger the event notification which will call the lambda function.
    * Lambda will read the file, create a corresponding HTML tag delimited test file and stores it in the caption-in file.
    * This will invoke another lambda function which calls the AWS translate service to convert the HTML tag delimited file to the language of choice.
    * AWS translate job place the translated file in the Amazon S3 bucket and emits an Amazon event bridge event when it is done. Event bridge call another lambda function.
    * This lambda function will read the delimited files from Amazon S3, creates the captions file in the required format (WEBVTT and SRT) and stores them in the output folder of the bucket.</br>
**Note: - Code and solution for translation is taken from the below mentioned stack. I have only included it in my overall solution or flow. Though this template is now present in my repository as well.**</br>
https://s3.amazonaws.com/aws-ml-blog/artifacts/translate-captions-files/v2/translate-captions-template-cf.yml
3.	**MediaConvert bucket:** As soon as the file is generated in the “output” folder of the Translate bucket, it will trigger another lambda function.
     * Lambda function will copy the generated captions file into the “mediacaptions” folder of the MediaConvert bucket.
     * Lambda function will copy the original video file from the “transinput” folder of the Transcribe bucket to the “mediavideos” folder of the MediaConvert bucket.
     * MediaConvert bucket creates a job in the AWS Elemental MediaConvert services to burn-in the translated captions with the original video.

## Enhancements:
1.   I have not written the delete object code for any bucket. Ideally it should be there to avoid the duplicity.
2.   Its better to use some database to track which caption file is tied to which video file. For now, I have just written a hack using “-” as delimiter to identify the video file.
3.   Step function can be used for the overall tracking of the jobs.
4.   Code for error handling is not very extensive, though cloudwatch logs are used for each of the services.

## Implementation steps
1.	Log in to your AWS account and access Cloudformation in the AWS console.
2.	Deploy the translation job using the cloudformation template placed in Subtitles-translate-job/06_translate-captions-template-cf.yml
* StackName: translation-caption-job
* Parameters:
 
![Alt Image text](/Images/translation-paremeter.png?raw=true "translation-job")

**Note: I have marked one parameters in Red as you will not see them in the initial template, later we will modify this template to include notifications for both transinput and transoutput folder, there we will include this parameters for importing variables from other stacks. This is to avoid circular dependencies of event notifications.**
**Note: In case, you are changing the destination language from hindi to something else, then you need to modify the json configuration file for mediaconverter as well.**

* Resources:
     * * AWS S3 bucket for translation
     * * IAM Policies and Roles for translation service and lambda
     * * Event Rule
     * * Lambda function
     * * Custom S3 Folders

3.	Next create the transcribe buckets using the cloudformation template placed in Subtitles-transcribe-job/01_transcribe-s3bucket.yaml. We will include event notifications in the same template later on and re-run it to avoid the circular dependencies.
 * Stackname: Subtitles-transcribe-bucket
 * Parameters

![Alt Image text](/Images/transcribe-bucket-parameter.png?raw=true "transcribe-bucket")
 
 

**Note: I have marked two parameters in Red as you will not see them in the initial template, later we will modify this template to include notifications for both transinput and transoutput folder, there we will need both of these parameters for importing variables from other stacks. This is to avoid circular dependencies of event notifications.**
 * Resources:
     * * AWS S3 bucket for transcribe
     * * Custom S3 folders
     * * IAM roles for Lambda custom function
	
4.	Next, we will create the lambda function to trigger the transcribe service. This function will be responsible for creating the transcribe jobs for extracting the speech. Templates for the same are included in Subtitles-transcribe-job/02_transcribe_lambda_job.yaml
* StackName: Subtitles-transcribe-lambda
* Parameters: 

![Alt Image text](/Images/transcibe-lambda-job.png?raw=true "transcribe-lambda-job")


* Resources:
     * * IAM Roles and permissions for the Lambda
     * * Lambda function to read from the transcribe bucket and trigger transcribe service
5.	Now, we will create a lambda function to copy extracted text file in the vtt format from transcribe bucket to the translate bucket. Templates are placed in Subtitles-transcribe-job/03_Copy_from_transcribe_to_translation.yaml.
* StackName: Subtitles-S3Copy
* Parameters:
![Alt Image text](/Images/transcribe-to-translate-bucketcopy.png?raw=true "bucketcopy")

 
* Resources:
     * * AWS IAM Roles and policy for the lambda function
     * * Lambda function to trigger the copy from transcribe bucket to the translate bucket
6.	Now we will once again update the Subtitles-transcribe-bucket created in 3rd step. Template for the same are placed in Subtitles-transcribe-job/05_transcribe-s3bucket.yaml. This will add two more parameters.
* Parameter 
 ![Alt Image text](/Images/6_transcribe-s3bucket.png?raw=true "transcribe-s3bucket")



7.	Next, mediaconvert buckets will be created. Template for the same is placed in Subtitles-mediaconvert-job/07_mediaconvert-s3bucket.yaml.
* StackName: Subtitles-mediaconvert-bucket
* Parameters:

 ![Alt Image text](/Images/7_mediaconvert-bucket.png?raw=true "mediaconvert")

* Resources
     * * AWS S3 bucket for mediaconvert
     * * Custom Folders for AWS S3
     * * AWS IAM Roles and policies for Lambda function to copy from one bucket to another.

8.	Only manual step in this implementation is to create one bucket and push the python code zip to that bucket and make it available to lambda function. Code can be found at Subtitles-mediaconvert-job/mediaconvert_lambda.zip.
9.	Once done, we will write the mediaconvert copy function and the mediaconvert job creation in the same function. Template for the same is placed in Subtitles-mediaconvert-job/08_mediaconverter-lambda-job-function.yml.
* StackName: Subtitles-Mediaconvert-copy
* Parameters: 
![Alt Image text](/Images/Mediaconvert-copy.png?raw=true "MediaConvertcopy") 
**Note: For destination languages other than Hindi, please use "mediaconvert_data_set['Settings']['OutputGroups'][0]['Outputs'][0]['CaptionDescriptions'][0]['LanguageCode']" to substitute the destination language.**

10.	Now, we will update the translation-caption-job template so that event notification can be added as soon as the translated vtt file hits the “output” folder it will be copied to the mediaconvert bucket. Template for the same is placed in Subtitles-translate-job/09_translate-captions-template-cf.yml.
* StackName: translation-caption-job
* Parameters
 
![Alt Image text](/Images/translation-parameter-final.png?raw=true "MediaConvertcopy")

Put the video.mp4 file in the “transinput” folder of the transcribe bucket and output will be generated in the “mediavideo” bucket of the mediaconvert bucket. It may take up to 10 to 15 minutes.
