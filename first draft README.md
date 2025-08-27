# MVC_Cluster
Cluster Resources reposeroty for Develep's submission projet
this repo hase the files for the configration and set up of the cluster

app-config 
creates confige map with data of enc, vaseurl and db

app-deloy 
created deployment thec creats pods of the application by 2 replica sets  for abilty ro do rolling updates for the pods 
uses the image we have in AWS ECR
exsposes ports localy to the cluster
runs some health cehks 
and limits resorses

app-svc 
creates loadbalance service thet exsposes port 80 and lisins to port 8000

mongo-values
sets the values for the halm cart for mango db 
sets replica number for the replicaset
uthentications an user configerions (i think i suled set it as secrets but that will be good for now)
exposes ports 27017
lmits resorses

mvc-secrets
sets up secrets for db uri that will be uses in the api