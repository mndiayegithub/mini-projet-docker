# student-list project
To find the specifications, please click on the following [link](https://github.com/diranetafen/student-list "link")

![project](https://user-images.githubusercontent.com/18481009/84582395-ba230b00-adeb-11ea-9453-22ed1be7e268.jpg)



------------
Name : NDIAYE 

Username: Mansour

Eazytraining's DevOps bootcamp 

Made the 23rd March 2023

------------

## Context 
POZOS is an IT company located in France and develops software for High School. The innovation department want to disrupt the existing infrastructure to ensure that it can be scalable, easily deployed with a maximum of automation. 

POZOS wants you to build a POC to show how docker can help you and how much this technology is efficient. For this POC, POZOS will give you an application and want you to build a "decouple" infrastructure based on Docker. Currently, the application is running on a single server with any scalability and any high availability. When POZOS needs to deploy a new release, every time some goes wrong.

In conclusion, POZOS needs agility on its software farm.

## Current application

The company wants to build an application able to show student list with their ages. 

The current application has two modules : 
- A REST API (with basic authentication needed) who send the desire list of the student based on JSON file
- A web app written in HTML + PHP who enable end-user to get a list of students

## Objectives

The objectives of this project are : 
- Build a docker based environment to improve the existing application into deployment,
- Add versioning to the infrastructure released,
- Address best practice when implementing docker infrastructure
- Build Infrastructure As Code

## Specifications
My work has to :
- Build one container for each module
- Make containers created to interact each other
- Provide a private registry to deploy the image built to the delivery

## Action plan
### Files used and their role
In the delivery, you will have three files : **Dockerfile**, **docker-compose.yml** and **docker-compose_registry**.yml.
- docker-compose.yml : To launch the application API and Website
- docker-compose-registry.yml : To launch the local registry and its frontend
- simple_api/student_age.py : Contains the source code of the python API
- simple_api/student_age.json : Contains student name with age and JSON format
- index.php : PHP page for the user interface with the service to list students with their age
- simple_api/Dockerfile : Build the API image based with the source code and personnalised parameters

### Build and test
1. Dockerfile to build the API image : 
```
FROM python:2.7-stretch
MAINTAINER mndiaye (mndiayepro97@gmail.com)
RUN apt-get update -y && apt-get install python-dev python3-dev libsasl2-dev python-dev libldap2-dev libssl-dev -y && \
    pip install flask==1.1.2 flask_httpauth==4.1.0 flask_simpleldap python-dotenv==0.14.0
COPY * /
VOLUME /data
EXPOSE 5000
CMD [ "python", "./student_age.py" ]
```

2. Clone the current application project and add Dockerfile into it :

```
git clone https://github.com/diranetafen/student-list.git
cd student-list/simple-api/
docker build . -t student_list_api.img
docker images 
```
 
![image](https://user-images.githubusercontent.com/58290325/227361927-6702f18a-880b-40f7-9c62-0c6323c44503.png)

3. Create a bridge-type network for the two containers to interact each other by their names :
```
docker network create student_list_network --driver=bridge
docker network ls
```

![image](https://user-images.githubusercontent.com/58290325/227365330-ff4565fe-7a8c-420e-83db-356ea6de1c08.png)

4. Run the API image as a container :
```
docker run --rm -d --name=student_list_api --network=student_list_network -v ./simple_api:/data student_list_api.img
```

![image](https://user-images.githubusercontent.com/58290325/227372635-1fb06593-a572-4229-82d9-dcc172beb4ff.png)

Firstly, on the built image, port 5000 was already exposed so I didn't had to add port argument on the command. 

Secondly, I mounted the volume ./simple_api on the local directory to the /data of the internal container, so that the JSON file student_age.json will be accessible inside the api container. 

5.  Update the `index.php` file :

You need to update the following line before running the website container to make **api_ip_or_name** and **port** fit your deployment  `$url = 'http://<api_ip_or_name:port>/pozos/api/v1.0/get_student_ages'`

Given that a bridge network permit two containers to interact each other via names, we can easily use the api container's name to make the API accessible by the website.

![image](https://user-images.githubusercontent.com/58290325/227373377-8dd3f7e9-4672-43be-82c1-3688375968fc.png)


6. Run the frontend webapp container :
```
docker run --rm -d --name=webapp_student_list -p 80:80 --network=student_list_network -v ./website:/var/www/html -e USERNAME=toto -e PASSWORD=python php:apache
docker ps
```

7. Test the API through the website : 
- By using command line : 
```
docker exec -it webapp_student_list curl -u toto:python -X GET http://student_list_api:5000/pozos/api/v1.0/get_student_ages
```

![image](https://user-images.githubusercontent.com/58290325/227376291-2492b80b-b533-4f9c-8bf7-83069c76d363.png)

- By using a web browser :

For example, I'm using VirtualBox VMs, so I can add port forwarding option via VirtualBox to be able to access the Website.  

![image](https://user-images.githubusercontent.com/58290325/227376786-58b501c2-52d9-4c7c-991e-3fc7701e0124.png)

Now, I can access the website on `http://127.0.0.1:8080` directly on my local machine. 

![image](https://user-images.githubusercontent.com/58290325/227377019-92ccae34-7b60-46fa-a2ae-15dfe154621e.png)

8. Clean the workspace

Each when work has been done successfully, we can clean the workspace. Thanks to the `--rm` argument used while creating containers, they will be removed automatically when they stop.

```
docker stop student_list_api
docker stop webapp_student_list
docker network rm student_list_network
```

### Deployment

Given that the **Build and Test** part has been successfully, we can now composerize our infrastructure into a `docker-compose.yml` file.
1. Write the docker-compose file and run the application (api + webapp)
```
docker-compose -f docker-compose.yml up -d
```

In the docker-compose file, I added an option to permit which container must start first. In this case, the API container must start first because the webapp depends on it.

![image](https://user-images.githubusercontent.com/58290325/227383490-26e0254e-7d36-4e5b-847b-a48e63ab1105.png)

Now, the application works :

![image](https://user-images.githubusercontent.com/58290325/227384210-5b519264-2be0-437b-98f6-b55bfcb651c2.png)


2. Create a registry and its frontend

To create a private registry, I used `registry:2` image and `joxit/docker-registry-ui:static` for its frontend GUI and passed some environment variables :

![image](https://user-images.githubusercontent.com/58290325/227385041-616b3aee-4583-4c07-9762-be3f719b7f0a.png)

```
docker-compose -f docker-compose_registry.yml up -d
```

![image](https://user-images.githubusercontent.com/58290325/227390010-8cfc7585-8055-44bb-9e39-3c3a38151851.png)


3. Push the image and test the GUI

We have to tag and push the image as below (`:latest` is optional) : 

```
docker tag student_list_api.img:latest pozos-registry:5000/pozos/student_list_api.img:latest
docker push pozos-registry:5000/pozos/student_list_api.img:latest
```

![image](https://user-images.githubusercontent.com/58290325/227391367-65d87341-1c60-41d5-9a1e-2983bacccfd8.png)

![image](https://user-images.githubusercontent.com/58290325/227391427-2b64478b-dd07-4b76-a80e-b5fde4b81000.png)

![image](https://user-images.githubusercontent.com/58290325/227391491-8fd91451-9912-4cd0-8cce-0bc948ffc160.png)

Here is the delivery of my Docker mini-project, hope you enjoy it !

![image](https://user-images.githubusercontent.com/58290325/227391740-d54a1b53-b197-40b9-9b13-fce07b706e1f.png)
