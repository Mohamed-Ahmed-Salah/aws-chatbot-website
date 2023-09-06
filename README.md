
https://github.com/Mohamed-Ahmed-Salah/aws-chatbot-website/assets/90526765/6263b7d3-3e88-428a-89cf-3331644ed814
# Static Website

The goal of this website is to support the learning of the Building Serverless Applications course.

This folder is designed to be downloaded locally and then uplaoded as is to S3, strictly following the instructions in the excercise guide.,


# Step 1 Create a chatbot
Create a chatbot with IAM role AWSServiceRoleForLexBots
Create intent for the weather and add slots for city

# Step 2 Create an S3 
Create an S3 bucket and upload the files and folders make it public and add the following policy : 
<!-- {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AddPerm",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::2019-03-01-er-website/*"
            ]
        }
    -->
note: make sure cache-control max-aga=0
# step 3 Create a cloud front distribution with the bucket name of the S3 website bucket.
Note bucket policy will update automaticly.
Select Redirect Http to Https and allowed http methods to GET, Head.
Default root object text.html

update S3 bucket policy
<!-- {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "2",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity XXXXXXXXX"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::Website-bucket/*"
        }
    ]
} -->

# Step 4 Create API GATEWAY
Choose REST protocol as a new API
Create method post as a mock (will update later)
update intergration response with 
<!-- {
    "temp_int": 69
} -->
Enable CORS and select Default 4xx and 5xx then deploy
take that api gateway url past it in s3website scripts/config.js
<!-- var API_GATEWAY_URL_STR = "https://Your-Invoke-URL.execute-api.us-east-1.amazonaws.com/test"; -->


# Step 5 Create AWS Lambda Funtion
create function with name "get_weather"
create a role from aws policy templates and name it Get-Weather 
choose "Basic Lambda@Edge Permissions(For CloudFront trigger")
Updtae your index.js to get city name and return temp
<!-- 
function handler(event, context, callback){    
     var 
         city_str = event.city_str,
         response = {
             city_str: city_str,
             temp_int: 74
         };
     console.log(response);
     callback(null, response);
 }
 exports.handler = handler;
 -->

 Replace API mock endpoint to get_weather lambda function 
 Make sure you enable CORS again and deploy and test weather bot again

# Step 6 Create a DynamoDB table
Primary key should be sc (Searchable City)
make read/write capacity 100 (Will change to 1 later)
create lambda function called seedDynamo to upload indexes to our table 
choose the existing role Get-Weather and add additional permissions to read list and write to DynamoDB 
create cities.csv in the enviroment of lambda
copy cities.md content from the s3website project and paste it in 
cities.csv
update index.js to:
  <!-- exports.handler = function(event, context, callback) { 
      var 
      	AWS = require("aws-sdk"),
      	fs = require("fs"),
        item = {},
      	some_temp_int = 0,
  		  params = {},
        DDB = new AWS.DynamoDB;
      
      AWS.config.update({
          region: "us-east-1"
      });
      fs.readFileSync("cities.csv", "utf8").split('\n').map(function(item_str){
          params.ReturnConsumedCapacity = "TOTAL";
          params.TableName = "weather";
          params.Item = {
              "sc": {
                  "S": item_str.split(",")[0]
              },
              "t": {
                  "N": String(item_str.split(",")[1])
              }
          };
          DDB.putItem(params, function(err, data){
              if(err){
                  console.error(err);
              }else{
                  //ignore output
              }
          });
      });
   setTimeout(function(){
       callback(null, "ok");
   }, 1000 * 10);
  } -->

  click run and check dynamoDB table.
  update DynamoDB capacity to 1
  delete the lambda function seedDyno
  update the get_weather lambda function index js to query DynamoDb and change time out to 15 secs:
<!-- 
  function handler(event, context, callback){
    var 
        AWS = require("aws-sdk"),
        DDB = new AWS.DynamoDB({
            apiVersion: "2012-08-10",
            region: "us-east-1"
        }),
        
        city_str = event.city_str.toUpperCase(),
        data = {
            city_str: city_str,
            temp_int_str: 72
        },
        response = {},
        params = {
            TableName: "weather",
            KeyConditionExpression: "sc = :v1",
            ExpressionAttributeValues: {
                ":v1":{
                    S: city_str
                }
            }
        };
    
   	DDB.query(params, function(err, data){
       var
       		item = {},
           	response = {
            	statusCode: 200,
            	headers: {},
            	body: null
        	};
        if(err){
            response.statusCode = 500;
            console.log(err);
            response.body = err;
        }else{
            // console.log(data.Items[0]);
            var data = data.Items[0];
            if(data && data.t){
                console.log(data.sc.S + " and " + data.t.N);
            	item = {
                    temp_int:Number(data.t.N),
                    city_str: data.sc.S
            	};
            }else{
                item = {
                	city_str: event.city_str
                  //when we don't return a temp, the client can say city not found
            	};
            }
        }
        response = item;
       // console.log(response);
        callback(null, response);
    });
}
exports.handler = handler;  -->



https://github.com/Mohamed-Ahmed-Salah/aws-chatbot-website/assets/90526765/31e9c7d2-067e-411b-829a-cf1f88f79dab


