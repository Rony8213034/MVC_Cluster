MVC_Cluster

Cluster resources repository for Develeap's submission project

This repo has the files for the configuration and setup of the cluster.

app-config

This file creates the configuration map for the application.

Creates ConfigMap with data for env, baseurl, and db

app-deploy

This file defines the application deployment.

Creates a Deployment that runs the application pods with 2 replicas for rolling updates

Uses the image stored in AWS ECR

Exposes ports locally to the cluster

Runs health checks

Limits resources

app-svc

This file manages the application service.

Creates a LoadBalancer service that exposes port 80 and listens to port 8000

mongo-values

This file holds the values for the MongoDB Helm chart.

Sets the values for the Helm chart for MongoDB

Defines replica number for the ReplicaSet

Handles authentication and user configuration (should probably be set as secrets, but left as-is for now)

Exposes port 27017

Limits resources

mvc-secrets

This file manages the cluster secrets.

Sets up secrets for DB URI that will be used in the API