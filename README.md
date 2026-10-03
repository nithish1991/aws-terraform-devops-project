# aws-terraform-devops-project

Overview
Creating EKS, VPC & Remote Backend Resources on AWS using Terraform

Terraform 
IAC tool which is vendor neutral to used to create infrastructure as code.


what is terraform statefile ?
It is file which holds the change log of terraform resource executions and act as the memeory for terreform. when a resoruce is created using terraform a records is added to the statefile and when a resoruce is deleted via terraform the resource is removed from the statfile. 

terraform statefile management
It is a vital component of terraform and it ideal to store in secured and centerized remote backend such as s3 in order to use terraform within in team. As storing the terraform statefile in the local machine will provide another team member with no idea of the what is created with the terraform file and when he trys to execute the same terrform file another set of resoruce will be created as the statfile was not on his local machine and this makes duplicate infrasture for the team.  


statefile locking

A mechanism to restrict to use terrform state file similateously by the team. Enabling statefile locking for the statefile will allow only one user to exacute the terraform file. 
Dynamodb is used to implement statefile locking 
s3 is used the remote backend to store the statefile

This repository contains Infrastructure as Code (IaC) for deploying a production grade DevOps Project Demo application on AWS.
It provisions S3 as a remote backend for storing the terraform statefile and locking mechanism, and creating EKS Cluster and VPC for the application.
