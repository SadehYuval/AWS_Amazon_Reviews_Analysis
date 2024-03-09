Gilad Rozner 318317708
Daniel Hayon 206520553
Yuval Sadeh 207466509

Instructions on how to run the project: (In VS Code as it is the platform in which we developed).

1 - Update your current AWS lab credentials in ~/.AWS/credentials using the nano in a Linux environment.
2 - At each of the folders in the project, app_project, manager_project and worker_project, run the command: mvn clean package.
3 - Take the created files named manager-jar-with-dependencies.jar and worker-jar-with-dependencies.jar, 
	which are located in the target folder in their respective project folders.
	move them to the app_project folder and rename them to manager.jar and worker.jar accordingly.
4 - Add all 5 of the input files to the app_project folder aswell, They can be found in the course moodle page under Assignment 1.	
5 - As for the app-jar-with-dependencies.jar files, it is sitting in the app_project target file, move outside of it to app_project.
6 - Use the command on the next line or adjust it as desired, depending on the amount of text files you wish to analyze:
	java -jar app-jar-with-dependencies.jar input1.txt input2.txt input3.txt input4.txt input5.txt output1.html output2.html output3.html output4.html output5.html 3 terminate

We ran the program using the following specs:
	We used the 'ami-00e95a9222311e8ed' ami as provided by the teaching assitant with T2_MEDIUM for the Manager EC2 instance 
	and T2_LARGE with an added Xmx4g memory for the Workers EC2 instances.
	We selected n=3 to reach a maximum of 8 workers due to AWS learnes lab constrictions.
	Under these conditions, after an uploading time of approximately 7 minutes of the large jar files to S3 the program took 18 miunutes to return outputs for all 5 files.

The flow of the program is as follows:

Local app:
1) Handle command line arguments.
2) Create an S3 bucket and upload input files, as well as manager and worker JARs if not already uploaded.
3) Create an EC2 instance for the manager if necessary.
4) Establish two SQS queues: one to send messages to the manager and the other to receive output from the manager.
5) Wait for the manager instance to be created, then send the local ID, file name, and bucket names to the manager through the respective SQS.
6) Await an output path from the manager, download the corresponding file from the S3 bucket, and create an HTML object using the file content.
7) Wait for a termination message; upon receiving it, terminate all running instances, buckets, SQS queues, and exit the program.

Manager:
1) Create an SQS to send messages to worker instances and another SQS to receive processed reviews from workers. Connect to the SQS created by the local app.
2) The manager operates as a thread listening to events from the local app and running workers, sleeping when idle and waking up on incoming messages from SQS.
3) Parse files into lines, treating each line as JSON, and distribute JSON segments to workers for processing.
4) Download the JAR file and launch the calculated number of required workers.
5) Upload results to S3 and send a message to the local app.
6) Terminate worker instances and the created SQS queues.
7) Send a "terminate" message back to the local app through its output SQS queue.

Worker:
1) Connect to the relevant SQS's created by the Manager.
2) Run in a loop, reading messages from the reviews SQS, and analyze them using the Stanford library.
3) Create a review using the Review class.
4) Process the review into JSON format.
5) Convert it to a string.
6) Send the processed review through the output SQS to the manager.	

The Program includes the properties:

Scalabilty - We designed the program to support multiple activations of the local application in parallel. 
			To distinguish between the activated local applications we genererated a unique ID for each of them so that the Manager
			saves in a hash table, which also contains relevant information about the actual input.txt files any certain local app sent for analysis.
			Using the data from this Hash Table, the Manager can return appropriate outputs to the relevant local app. 
			Following the formula of the number of reviews in need of processing in relation with the provided n the Manager generated Worker instances
			accordingly, up to the limit of 8 workers as we discovered 9 to be the limit amount of instances allowed to create under the AWS learners lab limitations.


Security - At no point in the code the credentials are exposed.

Threads - Our manager uses two threads to complete its actions, as we wished to support the parallel receipt of messages both from the local applications
		  sending more tasks for the program to complete, and from the workers replying to the requests of the Manager with the processed messages at the same time
		  as to not delay from the local apps to get output that are already ready to deliver by the Manager due to the manager being occupied by other tasks.

Crash Cases - By using visibilityTimeout as messages are received by the Worker instances and deleting them from the SQS solely after the 
			  worker has finished processing the message it pulled from the SQS, we made sure no data will be lost in case a worker will Crash
			  during the analysis of a message. 
Termination - The termination process is as such:
			  In case a local application included a final 'terminate' argument in the command activating the program, after the usual data, a termination message will be sent to the Manager,
			  which in turn shifts to termination mode, under this certain mode it stops listening to the messages sent via the SQS connecting it to the local application,
			  it is a fifo queue to make sure the messages will be sent and received in the right order so no previous data will be missed and no new data will be worked on.
			  Now as the Manager is in termination mode he waits for all the workers to finished their assigned jobs, we follow the messages pending for analysis
			  by maintaining coutners in our Hash Table, for each pair of correlating input-output files, said counters are increased as a message from the input files is sent
			  to the workers and decreased as a message returns processed to the manager and added to the output file.
			  Once all the work is done and outputs are sent back to the local apps, the manager shuts down all the running workers and closes the SQS's connecting it to the workers
			  and sends the local app that requested the termination a confirmation it indeed terminated.
			  Now the local app will shut the manager down, close the remaining SQS's and deletes the bucket and its content and exit cleanly.


General notes about our system:
	We tried to distrubute the work done by our program evenly, so no part is overused. By our logic the workers have it the hardest as they need to use
	the Standford package to analyze the text given to them, for this reason it is all they do, recieve a message, process it, send it back processed.
	The Manager has a heavy workload both downloading and uploading files so we split it into two threads to speed up the program.
	As far as we can tell, every link in the chain of our code has a defined role and it fulfills it proficiently with the use of the AWS services. 
	And with all the links working in unison we hope we developed an efficient program, answering the demands of the project.




			


 
	





