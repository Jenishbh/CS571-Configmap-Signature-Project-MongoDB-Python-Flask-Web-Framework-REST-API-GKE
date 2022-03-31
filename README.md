# CS571-Configmap-Signature-Project-MongoDB-Python-Flask-Web-Framework-REST-API-GKE
Creator 
Jenish Patel | 19549
Agenda
Step1: Create MongoDB using Persistent Volume on GKE, and insert records into it
Step2: Modify our studentServer to get records from MongoDB and deploy to GKE
Step3: Create a python Flask bookshelf REST API and deploy on GKE
Step4: Create ConfigMap for both applications to store MongoDB URL and MongoDB Name 
Step5: Expose 2 application using ingress with Nginx, so we can put them on the same Domain but different PATH



Notes
Please Install Minikube via. 
Install docker via 
You must have a docker id
Action Items
Step1: Create MongoDB using Persistent Volume on GKE, and insert records into it
Create a cluster as usual on GKE: 
gcloud container clusters create kubia --num-nodes=1 --machine-type=e2-micro --region=us-west1


Let’s create a Persistent Volume first:
gcloud compute disks create --size=10GiB --zone=us-west1-c mongodb

Now create a mongodb deployment with this yaml fileapiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  selector:
    matchLabels:
      app: mongodb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        # by defult, the image is pulled from docker hub
        - image: mongo
          name: mongo
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
      volumes:
        - name: mongodb-data
          gcePersistentDisk:
            pdName: mongodb
            fsType: ext4
 
 
 
 
And then create a deployment for the file
kubectl apply -f mongodb-deployment.yaml 

 
 
 
 
Check if the deployment pod has been successfully created and started running
Kubectl get pods

Create a service for the mongoDB, so it can be accessed from outside 
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  type: LoadBalancer
  ports:
    # service port in cluster
   - port: 27017
    # port to contact inside container
     targetPort: 27017
  selector:
    app: mongodb
 
 
And then create a deployment for the file
 kubectl create -f mongodb-service.yaml


Check the ip-address 
Kubectl get svc





Now try and see if mongoDB is functioning for connections using the External-IP
kubectl exec -it mongodb-deployment-replace-with-your-pod-name -- bash
Try 
mongo External-IP
 You should see something like this, which means your mongoDB is up and can be accessed using the External-IP






We need to insert some records into the mongoDB for later use 
node

To connect with your mongodb try following steps
var MongoClient = require('mongodb').MongoClient; 
var url = "mongodb://EXTERNAL-IP/mydb"

To add some records inside your database follow the below steps
MongoClient.connect(url,{ useNewUrlParser: true, useUnifiedTopology:true },function(err, client){
if (err)
throw err;
// create a document to be inserted
var db = client.db("studentdb");
const docs = [
{ student_id: 11111, student_name: "Bruce Lee", grade: 84},
{ student_id: 22222, student_name: "Jackie Chen", grade: 93 },
{ student_id: 33333, student_name: "Jet Li", grade: 88}
]
db.collection("students").insertMany(docs, function(err, res){
if(err) throw err;
console.log(res.insertedCount);
client.close();
});
db.collection("students").findOne({"student_id": 11111},
function(err, result){
console.log(result);
});
});

You should get the output like this










Step 2: Modify our studentServer to get records from MongoDB and deploy to GKE

Create a studentServer.js file
var http = require('http');
var url = require('url');
var mongodb = require('mongodb');
const {
MONGO_URL,
MONGO_DATABASE
} = process.env;
// - Expect the request to contain a query
// string with a key 'student_id' and a student ID as
// the value. For example
// /api/score?student_id=1111
// - The JSON response should contain only 'student_id', 'student_name'
// and 'student_score' properties. For example:
//
// {
// "student_id": 1111,
// "student_name": Bruce Lee,
// "student_score": 84
// }
//
var MongoClient = mongodb.MongoClient;
var uri = `mongodb://${MONGO_URL}/${MONGO_DATABASE}`;
// Connect to the db
console.log(uri);
var server = http.createServer(function (req, res) {
var result;
// req.url = /api/score?student_id=11111
var parsedUrl = url.parse(req.url, true);
var student_id = parseInt(parsedUrl.query.student_id);
// match req.url with the string /api/score
if (/^\/api\/score/.test(req.url)) {
// e.g., of student_id 1111
MongoClient.connect(uri,{ useNewUrlParser: true, useUnifiedTopology:
true }, function(err, client){
if (err)
throw err;
var db = client.db("studentdb");
db.collection("students").findOne({"student_id":student_id},
(err, student) => {
if(err)
throw new Error(err.message, null);
if (student) {
res.writeHead(200, { 'Content-Type': 'application/json'
})
res.end(JSON.stringify(student)+ '\n')
}else {
res.writeHead(404);
res.end("Student Not Found \n");
}
});
});
} else {
res.writeHead(404);
res.end("Wrong url, please try again\n");
}
});
server.listen(8080);
 





Create Dockerfile
FROM node:14
ADD studentServer.js /studentServer.js
RUN npm install mongodb
ENTRYPOINT ["node", "studentServer.js"]
 
 
 
 
 

 
 
 
 
 
 
 
 
 
 
 
 

Build the studentserver docker image

docker build -t yourdockerhubID/studentserver .


Push the docker image
If not then login the docker using 
docker login -u username -p password
		docker push yourdockerhubID/studentserver
		
	







Step 3 :Create a python Flask bookshelf REST API and deploy on GKE

. Create bookshelf.py 
 
from flask import Flask, request, jsonify
from flask_pymongo import PyMongo
from flask import request
from bson.objectid import ObjectId
import socket
import os
app = Flask(__name__)
 
app.config["MONGO_URI"] ="mongodb://"+os.getenv("MONGO_URL")+"/"+os.getenv("MONGO_DATABASE")
app.config['JSONIFY_PRETTYPRINT_REGULAR'] = True
mongo = PyMongo(app)
db = mongo.db
@app.route("/")
def index():
    hostname = socket.gethostname()
    return jsonify(message="Welcome to bookshelf app! I am running inside {}pod!".format(hostname))
@app.route("/books")
def get_all_tasks():
    books = db.bookshelf.find()
    data = []
    for book in books:
        data.append({
            "id": str(book["_id"]),
            "Book Name": book["book_name"],
            "Book Author": book["book_author"],
            "ISBN" : book["ISBN"]
        })
    return jsonify(data)
@app.route("/book", methods=["POST"])
def add_book():
    book = request.get_json(force=True)
    db.bookshelf.insert_one({
        "book_name": book["book_name"],
        "book_author": book["book_author"],
        "ISBN": book["isbn"]
    })
    return jsonify(message="Task saved successfully!")
@app.route("/book/<id>", methods=["PUT"])
def update_book(id):
    data = request.get_json(force=True)
    print(data)
    response = db.bookshelf.update_many({"_id": ObjectId(id)}, {"$set":{"book_name": data['book_name'],"book_author": data["book_author"], "ISBN": data["isbn"]}})
    if response.matched_count:
        message = "Task updated successfully!"
    else:
        message = "No book found!"
        return jsonify(message=message)
@app.route("/book/<id>", methods=["DELETE"])
def delete_task(id):
    response = db.bookshelf.delete_one({"_id": ObjectId(id)})
    if response.deleted_count:
        message = "Task deleted successfully!"
    else:
        message = "No book found!"
        return jsonify(message=message)
@app.route("/tasks/delete", methods=["POST"])
def delete_all_tasks():
    db.bookshelf.remove()
    return jsonify(message="All Books deleted!")
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)



Create a Dockerfile
FROM python:3.8-slim-buster
WORKDIR /app
RUN pip3 install flask
RUN pip3 install flask_pymongo
COPY . /app
ENV PORT 5000
EXPOSE 5000
ENTRYPOINT [ "python3" ]
CMD [ "bookshelf.py" ]

Build the docker image
docker build -t jenishbh/bookshelf

Push the docker image

docker push jenishbh/bookshelf



Step 4 :Create ConfigMap for both applications to store MongoDB URL and MongoDB name

Create a file named studentserver-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: studentserver-config
data:
MONGO_URL: Change-this-to-your-mongoDB-EXTERNAL-IP
MONGO_DATABASE: mydb


2. Create a file named bookshelf-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: bookshelf-config
data:
# SERVICE_NAME.NAMESPACE.svc.cluster.local:SERVICE_PORT
MONGO_URL: Change-this-to-your-mongoDB-EXTERNAL-IP
MONGO_DATABASE: mydb





















		
 
Step 5: Expose 2 application using ingress with Nginx, so we can put them on the same Domain but different PATH
Create studentserver-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: studentserver-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - image: jenishbh/studentserver
          imagePullPolicy: Always
          name: web
          ports:
            - containerPort: 8080
          env:
            - name: MONGO_URL
              valueFrom:
                configMapKeyRef:
                  name: studentserver-config
                  key: MONGO_URL
            - name: MONGO_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: studentserver-config
                  key: MONGO_DATABASE
 
 
Create bookshelf-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookshelf-deployment
  labels:
    app: bookshelf-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookshelf-deployment
  template:
    metadata:
      labels:
        app: bookshelf-deployment
    spec:
      containers:
      - image: jenishbh/bookshelf
        imagePullPolicy: Always
        name: bookshelf-deployment
        ports:
          - containerPort: 5000
        env:
          - name: MONGO_URL
            valueFrom:
              configMapKeyRef:
                name: bookshelf-config
                key: MONGO_URL
          - name: MONGO_DATABASE
            valueFrom:
              configMapKeyRef:
                name: bookshelf-config
                key: MONGO_DATABASE

Create sutdentserver-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  ports:
    # service port in cluster
    - port: 8080
      # port to contact inside container
      targetPort: 8080
  selector:
    app: web






. Create bookshelf-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: bookshelf-service
spec:
  type: LoadBalancer
  ports:
    # service port in cluster
    - port: 5000
    # port to contact inside container
      targetPort: 5000
  selector:
    app: bookshelf-deployment

Start minikube


Start Ingress
minikube addons enable ingress

Create studentserver related pods and start service using the above yaml file
kubectl apply -f studentserver-deployment.yaml
kubectl apply -f studentserver-configmap.yaml
kubectl apply -f studentserver-service.yaml
kubectl apply -f bookshelf-deployment.yaml
kubectl apply -f bookshelf-configmap.yaml
kubectl apply -f bookshelf-service.yaml


Check if all the pods are running correctly
kubectl get pods



Create an ingress service yaml file called studentservermongoIngress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: server
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: cs571.project.com 
      http:
        paths:
          - path: /studentserver(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
          - path: /bookshelf(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: bookshelf-service
                port:
                  number: 5000
Create the ingress service using the above yaml file
kubectl apply -f studentservermongoIngress.yaml

. Check if ingress is running
kubectl get ingress



Add Addreee to /etc/hosts
vi /etc/hosts
Add the address you got from above step to the end of the file
Your-address cs571.project.com
Your /etc/hosts file should look something like this after adding the line, but your address
should be different from mine



If everything goes smoothly, you should be able to access your applications curl cs571.project.com/studentserver/api/score?student_id=11111



On another path, you should be able to use the REST API with bookshelf application
I.e list all books
curl cs571.project.com/bookshelf/books



Add a book
curl -X POST -d "{\"book_name\": \"cloud computing\",\"book_author\":\"unkown\", \"isbn\": \"123456\" }" http://cs571.project.com/bookshelf/book




Update a book
curl -X PUT -d "{\"book_name\": \"123\",\"book_author\": \"test\", \"isbn\":\"123updated\" }" http://cs571.project.com/bookshelf/book/id





Delete a book curl -X DELETE cs571.project.com/bookshelf/book/id



