
# **Visit Logs Application**

This is a Node.js application that logs client information, including IP address, geolocation data, browser details, and visit timestamps, to a PostgreSQL database. It allows viewing logs in a paginated table format.


This guide explains how to deploy the application to **Google Cloud App Engine** with **Cloud SQL for PostgreSQL** and **Secret Manager** for secure configuration.

Visit Logs Application – Deployment Guide

This document explains how to deploy the Visit Logs Application to Google Cloud Platform using:

Google App Engine (Standard)

Cloud SQL (PostgreSQL)

Google Secret Manager

Google Cloud Build (CI/CD)



##Architecture Overview

App Engine hosts the Node.js web application

Cloud SQL stores visitor data in PostgreSQL

Secret Manager securely stores database credentials

Cloud Build automates build, migration, and deployment

GitHub triggers deployments on every push to main

1. Prerequisites

Before starting, ensure you have:

A Google Cloud account with billing enabled

Google Cloud CLI (gcloud) installed

A GitHub account

Node.js 22+ (for local testing only)

##2. Create a Google Cloud Project

Create a new project in the Google Cloud Console and note the Project ID.

Set the project as default:
```bash
gcloud config set project YOUR_PROJECT_ID

```
Enable billing for the project.

##3. Enable Required APIs

Enable all required Google Cloud services:
```bash
gcloud services enable appengine.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable secretmanager.googleapis.com
```
##4. Initialise App Engine

Create the App Engine application:
```bash
gcloud app create --region=europe-west2
```

You may choose another region, but ensure Cloud SQL is created in the same region.

##5. Create Cloud SQL (PostgreSQL)
5.1 Create the Cloud SQL Instance
```bashgcloud sql instances create visit-logs-postgres \
  --database-version=POSTGRES_17 \
  --region=europe-west2 \
  --tier=db-f1-micro
```
5.2 Create Database
```bash
gcloud sql databases create client_info \
  --instance=visit-logs-postgres
```
5.3 Create Database User
```bash
gcloud sql users create secure_user \
  --instance=visit-logs-postgres \
  --password=secure_password
```
##6. Store DATABASE_URL in Secret Manager
6.1 Get Cloud SQL Connection Name
gcloud sql instances describe 
```bashvisit-logs-postgres \
  --format="value(connectionName)"
```

Output example:

your-project-id:europe-west2:visit-logs-postgres

###6.2 Create Secret
```bash
echo -n "postgresql://secure_user:secure_password@/client_info?host=/cloudsql/YOUR_PROJECT_ID:europe-west2:visit-logs-postgres" | \
gcloud secrets create DATABASE_URL --data-file=-
```
###6.3 Verify Secret
```
gcloud secrets versions access latest --secret=DATABASE_URL
```
##7. Grant Secret Access to App Engine

Allow App Engine to read the secret:

gcloud secrets
```bashadd-iam-policy-binding DATABASE_URL \
  --member="serviceAccount:YOUR_PROJECT_ID@appspot.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```
##8. Configure app.yaml

Ensure the following file exists in the project root:
```bash
runtime: nodejs22
instance_class: F1
env: standard

automatic_scaling:
  max_instances: 2

env_variables:
  NODE_ENV: production
```
##9. Configure cloudbuild.yaml

Ensure the following file exists in the project root:

```bash
  # Step 1: Install dependencies
  - name: 'node:22'
    entrypoint: npm
    args: ['install']

  # Step 2: Run database migrations using Cloud SQL Proxy
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: bash
    secretEnv:
      - DATABASE_URL_MIGRATIONS
    args:
      - -c
      - |
        set -e

        echo "Installing Node.js..."
        apt-get update -qq
        apt-get install -y -qq nodejs npm curl ca-certificates > /dev/null

        echo "Downloading Cloud SQL Proxy..."
        curl -sSL -o cloud-sql-proxy \
          https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.14.2/cloud-sql-proxy.linux.amd64
        chmod +x cloud-sql-proxy

        echo "Starting Cloud SQL Proxy..."
        ./cloud-sql-proxy visit-logs-app:europe-west2:visit-logs-postgres --port 5432 &
        PROXY_PID=$$!

        sleep 5

        echo "Running migrations..."
        DATABASE_URL="$$DATABASE_URL_MIGRATIONS" npx sequelize-cli db:migrate

        echo "Stopping Cloud SQL Proxy..."
        kill $$PROXY_PID || true


  # Step 3: Deploy to App Engine
  - name: 'gcr.io/cloud-builders/gcloud'
    args: ['app', 'deploy', '--quiet']

availableSecrets:
  secretManager:
    - versionName: projects/visit-logs-app/secrets/DATABASE_URL_MIGRATIONS/versions/latest
      env: DATABASE_URL_MIGRATIONS

options:
  logging: CLOUD_LOGGING_ONLY

timeout: '900s'

```
##10. Push Code to GitHub

Create a GitHub repository and push the code:
```bash
git init
git add .
git commit -m "Initial deployment setup"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/visit-logs-app.git
git push -u origin main
```
##11. Create Cloud Build Trigger

Create a trigger that deploys on every push:
```bash
gcloud beta builds triggers create github \
  --name="deploy-visit-logs-app" \
  --repo-name="visit-logs-app" \
  --repo-owner="YOUR_USERNAME" \
  --branch-pattern="main" \
  --build-config="cloudbuild.yaml"
```
##12. Deployment Flow

From this point onwards:

Every push to main

Automatically triggers Cloud Build

Runs migrations

Deploys to App Engine

No manual deployment is required.

##13. Access the Application

Once deployed, the application is available at:

https://YOUR_PROJECT_ID.appspot.com


View logs:
```bash
gcloud app logs tail -s default
```
##14. Cleanup (Optional)

To avoid charges, delete the project:
```bash
gcloud projects delete YOUR_PROJECT_ID
```
##Summary

This deployment demonstrates:

Secure credential handling

Automated CI/CD

Cloud SQL integration

App Engine orchestration

Production-style deployment workflow

13. Live Deployment URLs

The deployed Application can be accessed through following links
Base URL

https://visit-logs-app.nw.r.appspot.com

Available Endpoints

Homepage

https://visit-logs-app.nw.r.appspot.com/


Displays application status and confirms successful deployment.

Client Information API

https://visit-logs-app.nw.r.appspot.com/api/client-info


Returns the visitor’s IP address, browser details, and geolocation data.

Visit Logs

https://visit-logs-app.nw.r.appspot.com/logs


Displays a paginated table of all recorded visits stored in Cloud SQL.

Health Check Endpoint

https://visit-logs-app.nw.r.appspot.com/_health


Used by Google Cloud for monitoring:

Returns 200 OK when the database is connected

Returns 503 Service Unavailable if the database is unreachable

