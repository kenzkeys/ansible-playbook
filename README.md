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
   ---
- name: Deployment Setup
  hosts: hng
  become: yes

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

  tasks:
    # Step 0: Update apt cache
    - name: Update apt cache
      apt:
        update_cache: yes

    # Step 1: Check if sudo group exists
    - name: Does sudo group exist
      command: getent group sudo
      register: sudo_group
      ignore_errors: true

    - name: If no, create sudo group
      group:
        name: sudo
        state: present
      when: sudo_group.rc != 0

    - name: Ensure sudo group has sudo privileges
      lineinfile:
        path: /etc/sudoers
        state: present
        regexp: '^%sudo'
        line: '%sudo ALL=(ALL:ALL) ALL'
        validate: '/usr/sbin/visudo -cf %s'

    # Step 2: Create deploy_user and grant it sudo privileges
    - name: Install whois package
      apt:
        name: whois
        state: present

    - name: Generate password hash
      command: "mkpasswd --method=SHA-512 {{ deploy_passwd }}"
      register: hashed_password
      no_log: true

    - name: Create deploy_user
      user:
        name: "{{ deploy_user }}"
        groups: sudo
        append: yes
        state: present
        shell: /bin/bash
        password: "{{ hashed_password.stdout }}"

    - name: Grant deploy_user sudo privileges without password
      lineinfile:
        path: /etc/sudoers
        state: present
        regexp: '^{{ deploy_user }}'
        line: '{{ deploy_user }} ALL=(ALL) NOPASSWD:ALL'
        validate: '/usr/sbin/visudo -cf %s'

    # Step 3: Create directories with proper ownership and permissions
    - name: Create log_dir directory
      file:
        path: "{{ log_dir }}"
        state: directory
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0755'

    - name: Create local_repo directory
      file:
        path: "{{ local_repo }}"
        state: directory
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0755'

    - name: Create postgres_cred_dir directory
      file:
        path: "{{ postgres_cred_dir }}"
        state: directory
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0755'

    # Step 4: Ensure files are present or create them with proper ownership and permissions
    - name: Ensure Log files are present
      file:
        path: "{{ item }}"
        state: touch
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0644'
      with_items:
        - "{{ log_stderr }}"
        - "{{ log_stdout }}"

    - name: Ensure postgres_cred file is present
      file:
        path: "{{ postgres_cred }}"
        state: touch
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0600'

    ######################## LOGGING STARTS ########################
    # Step 5: Ensure required python packages are installed
    - name: Ensure python3-venv is installed, if not install it
      apt:
        name: python3-venv
        state: present
      register: python3_venv_output

    - name: Ensure python3-psycopg2 is installed, if not install it
      apt:
        name: python3-psycopg2
        state: present
      register: python3_psycopg2_output

    # Step 6: Configure Git safe directory
    - name: Configure Git safe directory
      command: git config --global --add safe.directory "{{ local_repo }}"
      become_user: "{{ deploy_user }}"

    # Step 7: Ensure correct ownership of local_repo directory
    - name: Ensure correct ownership of local_repo directory
      file:
        path: "{{ local_repo }}"
        state: directory
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        recurse: yes

    # Step 8: Clone the git repository
    - name: Clone the git repository
      git:
        repo: "{{ git_repo }}"
        dest: "{{ local_repo }}"
        version: "{{ git_branch }}"
        force: yes
      become_user: "{{ deploy_user }}"
      register: git_output

    # Step 9: Create a virtual environment, activate it and install the required packages
    - name: Create virtual environment
      command: python3 -m venv "{{ local_repo }}/{{ venv }}"
      become_user: "{{ deploy_user }}"
      args:
        chdir: "{{ local_repo }}"
      register: venv_output

    - name: Activate virtual environment
      shell: |
        . {{ local_repo }}/{{ venv }}/bin/activate && echo $VIRTUAL_ENV
      become_user: "{{ deploy_user }}"
      args:
        executable: /bin/bash
      register: Activate_venv_output

    - name: Install dependencies from requirements.txt
      pip:
        requirements: "{{ local_repo }}/requirements.txt"
        virtualenv: "{{ local_repo }}/{{ venv }}"
      become_user: "{{ deploy_user }}"
      register: pip_output

    # Step 10: Set up PostgreSQL
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present
      register: install_postgres_output

    - name: Ensure PostgreSQL service is running
      service:
        name: postgresql
        state: started
        enabled: yes
      register: postgres_service_output

    - name: Create PostgreSQL database
      become_user: postgres
      community.postgresql.postgresql_db:
        name: "{{ postgres_db }}"
        state: present
      register: create_db_output

    - name: Create PostgreSQL admin user
      become_user: postgres
      community.postgresql.postgresql_user:
        name: "{{ postgres_admin_user }}"
        password: "{{ postgres_admin_password }}"
        db: "{{ postgres_db }}"
        priv: "ALL"
        state: present
      register: create_admin_user_output

    - name: Save PostgreSQL admin credentials to postgres_cred file
      copy:
        content: |
          DB_TYPE=postgresql
          DB_NAME={{ postgres_db }}
          DB_USER={{ postgres_admin_user }}
          DB_PASSWORD={{ postgres_admin_password }}
          DB_HOST="localhost"
          DB_PORT=5432
          DB_URL=postgresql://{{ postgres_admin_user }}:{{ postgres_admin_password }}@localhost:5432/{{ postgres_db }}
        dest: "{{ postgres_cred }}"
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0600'

    - name: Create a dummy table in PostgreSQL database
      become_user: postgres
      postgresql_query:
        db: "{{ postgres_db }}"
        query: "CREATE TABLE IF NOT EXISTS dummy_table (column1 VARCHAR(255), column2 VARCHAR(255));"

    - name: Create dummy data in PostgreSQL database
      become_user: postgres
      postgresql_query:
        db: "{{ postgres_db }}"
        query: "INSERT INTO dummy_table (column1, column2) VALUES ('dummy1', 'dummy2');"

    # Step 11: Create and update the .env file
    - name: Copy ".env.sample" to ".env"
      copy:
        src: "{{ local_repo }}/.env.sample"
        dest: "{{ local_repo }}/.env"
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0644'
        remote_src: yes

    - name: Update .env file with Postgres URL
      lineinfile:
        path: "{{ local_repo }}/.env"
        regexp: '^DB_URL='
        line: "DB_URL=postgresql://{{ postgres_admin_user }}:{{ postgres_admin_password }}@localhost:5432/{{ postgres_db }}"

    - name: Update .env file with Postgres details
      lineinfile:
        path: "{{ local_repo }}/.env"
        regexp: '^DB_NAME='
        line: "DB_NAME={{ postgres_db }}"

    - name: Update .env file with Postgres user
      lineinfile:
        path: "{{ local_repo }}/.env"
        regexp: '^DB_USER='
        line: "DB_USER={{ postgres_admin_user }}"

    - name: Update .env file with Postgres password
      lineinfile:
        path: "{{ local_repo }}/.env"
        regexp: '^DB_PASSWORD='
        line: "DB_PASSWORD={{ postgres_admin_password }}"

    # Step 12: Run the FastAPI application
    - name: Run FastAPI application
      shell: |
        . {{ local_repo }}/{{ venv }}/bin/activate
        nohup uvicorn main:app --host 0.0.0.0 --port {{ app_port }} &
      args:
        chdir: "{{ local_repo }}"
      become_user: "{{ deploy_user }}"
      async: 0
      poll: 0
      register: uvicorn_output

    # Step 13: Install Nginx
    - name: Ensure Nginx is installed, if not install it
      apt:
        name: nginx
        state: present
      register: nginx_output

    - name: Ensure Nginx service is running
      service:
        name: nginx
        state: started
        enabled: yes
      register: nginx_service_output

    - name: Remove default Nginx configuration
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

    - name: Configure Nginx to reverse proxy FastAPI application
      template:
        src: nginx_config.55
        dest: /etc/nginx/conf.d/default.conf
      notify:
        - restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

  # Logging in place with proper handlers for different steps
  post_tasks:
    - name: Log Step 6 Output
      local_action: copy content="{{ git_output.stdout | default('') }}" dest="git_output.log"
      when: git_output is defined and git_output.stdout is defined

    - name: Log Step 7 Output
      local_action: copy content="{{ venv_output.stdout | default('') }}" dest="venv_output.log"
      when: venv_output is defined and venv_output.stdout is defined

    - name: Log Step 8 Output
      local_action: copy content="{{ install_postgres_output.stdout | default('') }}" dest="install_postgres_output.log"
      when: install_postgres_output is defined and install_postgres_output.stdout is defined

    - name: Log Step 9 Output
      local_action: copy content="{{ copy_env.stdout | default('') }}" dest="copy_env_output.log"
      when: copy_env is defined and copy_env.stdout is defined

    - name: Log Step 10 Output
      local_action: copy content="{{ uvicorn_output.stdout | default('') }}" dest="uvicorn_output.log"
      when: uvicorn_output is defined and uvicorn_output.stdout is defined

    - name: Log Step 11 Output
      local_action: copy content="{{ nginx_output.stdout | default('') }}" dest="nginx_output.log"
      when: nginx_output is defined and nginx_output.stdout is defined
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
