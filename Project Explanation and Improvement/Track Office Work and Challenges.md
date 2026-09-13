### Challenges and Problems
1.  PPtx js Text overlap for japanese language . Takes 2 width of the size for each character 
2.  Blur functionality 
3.  Service->apigateway->sqs->lambda->opensearch 
4. Kinesis vs SQS
5. VPC in lambda and role where left . creata a role in opensearch as well and need to specify the index that it can handle 
6. Added too less time for the execution for the lambda and got troubled as it got executed ealry. Increased the active time for the lambda so that it executes for a good time.
7. Max retries count for qeues was 3 but seeing duplicate data in the dlq why?
8. Date format issue . Opensearch (Range queries and Avg time spent per day )
9. Ssl certs issue 
10. Role issue for the lambda . havent given putandpost _opensearch service policy 
11. Raise expection in lambda . then message will not get proceed and the since we are not raising any expection sqs might delete that mssg . and we will lost the data
12. Used S3 bucket . In python we were using some files. and in lambda there is a restriction of uploading large file . so first uplaod to s3 bucket and then import from s3 to lambda (faced while python library) . 
13. Used Opensearch py library first in the lambda that was costing us very much space .than after seeing the estimation i have changed that by normal requests library . 
14. Prod issues Rectified Logic for Excel, ppt, word, pdf. Timestamp as the key with description and the screenshot  .
15. prod issue for tag value in SAP value . 
16. shortcut key fix  for dff . Keyboard listener for sap 
17. Improve Ui Performance by using Use ref and passing the object as prop to children websites . In child i am fetching the data from elastic serach and processed the data into the useref. Improve performance as for ppt , exel , word i dont have to call the elastic again again ... 
18. Unit test Case using Jest , react testing Library . 
19. In APiGateway we had used AWS cognito with apigateway
20. Learnt about LOG rotation , file name as username-date . After some files logs files will get rotatate.
21. Authservice in Agent App. AWS cognito , ID token, refersh token, access token. 
22. Director wants to store the images on premise as the product is going to be sold to the finanicial client . Opensearch is a cloud depeency 
23. API Gateway has Limit of 10 mb not suitable to images so need ngnix load balancer . 
24. refersh TOken issue . When i am requesting the id token and access token i am not getting the id token and access token . Solution was -> refersh token doesnt mathc with with the client id 
25. Implemented Start/Stop recording feature in our Agent app . Used opensearch for Images storage . Designed robust idempotency api for our operations . 
26. Improve the api performace from 3s to 27 ms . IN start and stop apis . first what i was doing to start all the application on the start toggle and end all the application at the toggle off . But the api response was taking too much so i just make them running and only allow if the capture state is running  only .
27. Encounter CORS express . Api was not able to get acessed fom the site . But it was accesed throught postman. Used app.user(cors);
28. Logout feature in Agent app for Pa and agent user 
29. SSO Login and site is unsecure -> 
30. Nginix because of limitation of api gateway response limit of 10 mb . Implementing rate limiting in Ngnix and it act as a reverse proxy as well
31. Certs issue as chrome doesnt support sending request to local host . 
32. Proposed 2 solution > 1 to Inject the cert in Trusted CA in the system . 
33. 2 > Second way to implement this using the middle service . Agent will register itself to the middle service deployed to ECS  and there will be 2 way communication between middleservice <-> FE and middleService <-> Agent Registered 
34. Discussion happend on SSE and Websockets 
35. ALB usses http2 . API Gateway -> ALB -> Resource map -> Routing to targetGroups -> ECS service Trigger will start . This middle service is reponsible to maintain the socket mapping .  For now its in memory 
36. Api Gateway Limit of 20mb to get pass on . So Image Optimization in images . And also THinking of new Architecture to scale the no . 
37. Imges would be store on S3 bukcet / blob storage . And in OpenSearch there will be only URL stored . This will make the Overall Size of the response under 50kb . 
38. Worked ON Vapt Issue Login Issue . handle Concurent login . made us of node-cache . where it will be storing . Stored Email + Session id . Logic Allow latest Session and logout from the existing session . 
39. Handle Spamming of my start and stop button by addin a loader to it . 
40. Take ownership of this feature end to end and make it upto production . 
41. Take ownership of the Concurent login issue . Implemented the logic that with single credentials only one user can be login . Implemented the logic of the session id . Here In backend we have Session in the cache and whenever a email came up with new session id and the diff session id is already there then we would logout the old one. 
42. Wrote Logic of Bulk Insert in case of All Screen Capture . As we were getting 413 entity too large error from the lambda . Lambda has 6mb request / response size . Reduce Image size from 500kb to 50kb by converting it into jpeg from png and reducing the quality by 60% . Also Introduced . Also Decreased the Latency of this by reducing the lambda request and cost of aws . Right we were calling lambda for each request . But now i am sending data in form of the chunks of 3 . This reduced the Lambda calls and also the cost . 
43. Speech to Text Feature in DFF https://dev.to/devsmitra/speech-to-text-application-in-few-lines-of-javascript-and-html-3d3n
44. How Does App versioning  would work and also explain how can we show up the pop up for users to upgrade the app . At the time of login for each users store the version of the app itself. 
45. Added Status feature in the app . which will give us the status to be Intializing ... or ready . 
    Instead of polling of the exes continously. I am now polling only at the starting of the app . Once app is ready i stop the polling part . Hence optimized and performance of the app. 
46. Add App versioning . Basically all the app files would be maintained at S3 . There a route create to get the presigned url and the s3 key is present in the db . 
47. Each user app version would be stored in the db and there would be a source of truth as well which will contian the latest app version . I would need to work on the auto app updater to give users a seemless eperience . 
48. Opensearch Burst . When we burst out our open search we got 502 gateway / no server available for opensearch. Its because the images size and the over doc size gets so huge that we can process. 
49. solution to opensearch busrt was SCROll api to get data in chunks 
50. Our shards count became 3248 which was huge . we reduced that down and also deleted no of unnecessay indexes. 
51. Scroll Api for Getting data from elastic Search if i try to get the data in one go. it wont be able to query more than 100mb data So to get the data in limited fasion we will hit the Openseach with Scroll =1m for the first time and then hit the second batch of data with Scroll Id . 
52.  App Waa hosted on the S3 bucket and the bucket name and key was stored in the secret manager . This will install the auto update the app . 
53. used Electron squirrel to auto update the app . 
54. Use pupeter for PDF Genreration . Pdf genreation will failed for more than 2k slides so to make sure it would work we would use a appoach to make pdf in a batch of 25slides and then make them merge it . 
55. Implemented Agent Versioning . Introdced two api for checking the AgentVersion that will be store in the secret manager and the AgentInstall api that will get me the agent files downoad that is hosted at s3 bucket . for that we have bucket and key . 
56. In web app also when the user login we will check if the agent version for that user is good or not if its not good than show a pop up that Please upgrade your desktop application. 
57. Handle Pdf Generation logic . Instead of making complete pdf in one go . Added a chunking logic of making small pdf of sie 100 and then merge all of them in end . Making sure backend load minmal . Challenges to temper the template make the global index to handle it . 
58. Agent Version mai Api bani thi AgentVersion to get the version from Secret Manager . Routing ki thi for all this .
59. Red Box Logic in Complete Capture Logic. 
60. Shortcuts logic in outlook, SAP .
61. Found One more thing that payload size of Lambda is very less . Its 6mb if its direct and if its behind alb then its 1mb only not more than that . Thats why faced 413 error in the payload itself.
    
  