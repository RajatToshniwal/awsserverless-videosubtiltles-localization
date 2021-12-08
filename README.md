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
    * This lambda function will read the delimited files from Amazon S3, creates the captions file in the required format (WEBVTT and SRT) and stores them in the output folder of the bucket.
**Note:** - Code and solution for translation is taken from the below mentioned stack. I have only included it in my overall solution or flow. Though this template is now present in my repository as well.
https://s3.amazonaws.com/aws-ml-blog/artifacts/translate-captions-files/v2/translate-captions-template-cf.yml
3.	**MediaConvert bucket:** As soon as the file is generated in the “output” folder of the Translate bucket, it will trigger another lambda function.
     * Lambda function will copy the generated captions file into the “mediacaptions” folder of the MediaConvert bucket.
     * Lambda function will copy the original video file from the “transinput” folder of the Transcribe bucket to the “mediavideos” folder of the MediaConvert bucket.
     * MediaConvert bucket creates a job in the AWS Elemental MediaConvert services to burn-in the translated captions with the original video.

