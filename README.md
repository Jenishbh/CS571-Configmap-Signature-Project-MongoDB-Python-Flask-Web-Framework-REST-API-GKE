# CS571-Configmap-Signature-Project-MongoDB-Python-Flask-Web-Framework-REST-API-GKE

- Creator: Jenish Patel | 19549

#### Agenda: 
- Create MongoDB using Persistent Volume on GKE, and insert records into it
- Create a python Flask bookshelf REST API and deploy on GKE
- Create ConfigMap for both applications to store MongoDB URL and MongoDB Name 
- Expose 2 application using ingress with Nginx, so we can put them on the same Domain but different PATH
	

#### Run 2 Pods

- Pod 1 : Student server 
```
curl cs571.project.com/studentserver/api/score?student_id=11111
```
- ![aaa](https://user-images.githubusercontent.com/60593990/161001412-ebbd3d47-8e23-496c-9c71-06e24f38c706.png)


- Pod 2: Book Records

```
curl cs571.project.com/bookshelf/books
```

 ![sa](https://user-images.githubusercontent.com/60593990/161001642-c12516b1-6589-4f21-a4c4-74a505d4ae62.png)
