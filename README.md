# Spring Boot + MySQL Dockerized Application with Jenkins CI/CD Pipeline

## Overview

This project demonstrates how to Dockerize a Spring Boot application connected to a MySQL database, and how to automate its build and deployment using a Jenkins pipeline. The project includes:

- **Spring Boot application** packaged as a Docker container
- **MySQL database** running in a separate Docker container
- A shared Docker network for container communication
- A **Jenkins pipeline** that builds, deploys, and manages both containers intelligently

---

## Repository Contents

- `Dockerfile` — Builds Docker image for the Spring Boot app  
- `application.properties` — Configured for MySQL connection via Docker container hostname  
- `Jenkinsfile` — Pipeline script for CI/CD automation  
- Spring Boot source code and Maven project files

---

## Prerequisites

- Jenkins installed with Docker and Maven available on the agent node  
- Docker installed and running on the Jenkins agent machine  
- Git access to this repository  

---

## How It Works

1. **Docker Network Setup**  
   Jenkins pipeline creates a Docker network `app-network` if it does not already exist.

2. **MySQL Container**  
   The pipeline checks if a MySQL container named `mysql-container` is running; if not, it starts or creates it with root password `1234`.

3. **Build Spring Boot Application**  
   Runs Maven to clean and package the Spring Boot JAR without tests.

4. **Build Spring Boot Docker Image**  
   Uses the provided `Dockerfile` to build a Docker image tagged as `spring-app`.

5. **Run Spring Boot Container**  
   Checks if a container named `spring-app-container` is running. If not, it removes any stopped containers with the same name and runs a new container attached to the Docker network.

6. **Connectivity**  
   The Spring Boot app connects to MySQL via hostname `mysql-container` on port 3306.

---

## Configuration Details

### `application.properties`

```properties
spring.datasource.url=jdbc:mysql://mysql-container:3306/myapplication?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=1234
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

---
## admin login

- username: admin
- password: admin
---
*Script Done by SAK*