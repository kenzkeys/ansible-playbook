# Project Deployment with Ansible

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Prerequisites](#prerequisites)
4. [Setting Up the Environment](#setting-up-the-environment)
5. [Deployment Process](#deployment-process)
6. [Testing the Deployment](#testing-the-deployment)
7. [Troubleshooting](#troubleshooting)
8. [Conclusion](#conclusion)

## Introduction
This project demonstrates the automated deployment of a FastAPI application using Ansible. The deployment process includes setting up the necessary system configurations, creating a virtual environment, installing dependencies, configuring PostgreSQL, and setting up Nginx as a reverse proxy.

## Project Structure
The main components of the project are:
- `main.yaml`: The main Ansible playbook that outlines the deployment steps.
- `templates/nginx_config.j2`: The Jinja2 template for configuring Nginx.

## Prerequisites
Before running the deployment, ensure you have the following:
- Ansible installed on your local machine.
- Access to the target server with appropriate SSH permissions.
- The target server should have internet access to download packages and dependencies.

## Setting Up the Environment
1. **Clone the Repository:**
   ```bash
   git clone https://github.com/hngprojects/hng_boilerplate_python_fastapi_web.git
   cd hng_boilerplate_python_fastapi_web
   ```

2. **Install Ansible:**
   ```bash
   sudo apt update
   sudo apt install ansible -y
   ```

3. **Configure Inventory:**
   Create an inventory file (`hosts.ini`) and specify your target server(s):
   ```ini
   [hng]
   your_server_ip ansible_user=your_ssh_user
   ```

## Deployment Process
1. **Prepare the Ansible Playbook:**
   Ensure the `main.yaml` playbook is correctly configured with necessary variables:
   ```yaml
   vars:
     git_branch: "devops"
     git_repo: "https://github.com/hngprojects/hng_boilerplate_python_fastapi_web.git"
     local_repo: "/opt/stage_5b"
     venv: "venv"
     log_dir: "/var/log/stage_5b"
     postgres_cred_dir: "/var/secrets"
     log_stderr: "/var/log/stage_5b/error.log"
     log_stdout: "/var/log/stage_5b/out.log"
     postgres_cred: "/var/secrets/pg_pw.txt"
     deploy_user: "hng"
     deploy_passwd: "{{ deploy_user }}"
     app_port: 3000
     nginx_port: 80
     postgres_db: "database"
     postgres_admin_user: "admin"
     postgres_admin_password: "passwd"
   ```

2. **Run the Playbook:**
   Execute the Ansible playbook to deploy the application:
   ```bash
   ansible-playbook -i hosts.ini main.yaml
   ```

## Testing the Deployment
1. **Verify Nginx Configuration:**
   Ensure Nginx is running and the configuration is correct:
   ```bash
   sudo nginx -t
   sudo systemctl status nginx
   ```

2. **Check Application Logs:**
   Check the application logs to ensure it is running without errors:
   ```bash
   tail -f /var/log/stage_5b/out.log
   tail -f /var/log/stage_5b/error.log
   ```

3. **Access the Application:**
   Open a web browser and navigate to your server's IP address to access the application:
   ```
   http://your_server_ip
   ```

## Troubleshooting
- **Ansible Errors:** Use the `-vvv` option with `ansible-playbook` for detailed output.
- **Service Status:** Check the status of Nginx and PostgreSQL services:
  ```bash
  sudo systemctl status nginx
  sudo systemctl status postgresql
  ```
- **Log Files:** Review application logs for any errors:
  ```bash
  cat /var/log/stage_5b/error.log
  cat /var/log/stage_5b/out.log
  ```

## Conclusion
This documentation covers the setup, deployment, and testing of a FastAPI application using Ansible. By following these steps, you can automate the deployment process, ensuring consistency and reliability across different environments.

For any further questions or issues, please refer to the Ansible and FastAPI documentation or contact the project maintainers.
