# hng_boilerplate_golang_web

The provided README content is well-structured and includes detailed steps for deploying the `hng_boilerplate_golang_web` application using Ansible on an Ubuntu EC2 instance. Here are some minor adjustments and improvements to ensure compatibility with GitHub's markdown formatting and clarity:

```markdown
# hng_boilerplate_golang_web

### Detailed Deployment Procedure for `hng_boilerplate_golang_web` Application

This procedure will guide you through deploying the `hng_boilerplate_golang_web` application using Ansible on an Ubuntu EC2 instance, including necessary steps, configurations, and troubleshooting tips.

#### Prerequisites

1. **AWS EC2 Instance:** Ubuntu 22.04 or later.
2. **SSH Key Pair:** For accessing your EC2 instance.
3. **Ansible Installed:** On your local machine or a control machine.

### Steps to Deploy the Application

#### Step 1: Set Up the EC2 Instance

1. **Launch EC2 Instance:**
   - Launch an Ubuntu 22.04 EC2 instance.
   - Ensure the security group allows inbound traffic on ports 22 (SSH), 80 (HTTP), and 3000 (for the application).

2. **SSH into the EC2 Instance:**
   ```bash
   ssh -i /path/to/your-key-pair.pem ubuntu@your-ec2-public-ip
   ```

#### Step 2: Prepare the Ansible Playbook

1. **Create the Ansible Inventory File:**

   `inventory.cfg`
   ```ini
   [hng]
   your-ec2-public-ip ansible_user=ubuntu ansible_ssh_private_key_file=/path/to/your-key-pair.pem
   ```

2. **Create the Ansible Playbook:**

   `main.yaml`
   ```yaml
   ---
   - hosts: hng
     become: yes
     tasks:
       - name: Update and upgrade apt packages
         apt:
           update_cache: yes
           upgrade: dist

       - name: Create hng user
         user:
           name: hng
           shell: /bin/bash
           create_home: yes
           groups: sudo

       - name: Install required packages
         apt:
           name:
             - postgresql
             - nginx
             - golang
             - git
             - rabbitmq-server
           state: present
           update_cache: yes

       - name: Set up directory for PostgreSQL credentials
         file:
           path: /var/secrets
           state: directory
           mode: '0755'

       - name: Save PostgreSQL credentials
         copy:
           content: "admin_user: admin\nadmin_password: securepassword"
           dest: /var/secrets/pg_pw.txt
           mode: '0600'

       - name: Create PostgreSQL user
         command: sudo -i -u postgres psql -c "CREATE USER admin WITH PASSWORD 'securepassword';"
         ignore_errors: yes

       - name: Create PostgreSQL database
         command: sudo -i -u postgres psql -c "CREATE DATABASE hng_db OWNER admin;"
         ignore_errors: yes

       - name: Ensure /opt/stage_5b is a safe directory for Git
         command: git config --global --add safe.directory /opt/stage_5b

       - name: Clean the existing repository directory
         file:
           path: /opt/stage_5b
           state: absent

       - name: Clone repository as root
         git:
           repo: 'https://github.com/hngprojects/hng_boilerplate_golang_web'
           dest: /opt/stage_5b
           version: devops
           accept_hostkey: yes

       - name: Change ownership to hng user
         file:
           path: /opt/stage_5b
           state: directory
           owner: hng
           group: hng
           recurse: yes

       - name: Create and populate .env file
         copy:
           content: |
             export SERVER_PORT=3000
             export SERVER_SECRET="replace_with_strong_unique_secret_key"
             export SERVER_ACCESSTOKENEXPIREDURATION=2
             export REQUEST_PER_SECOND=6
             export TRUSTED_PROXIES="192.168.0.1,192.168.0.2"
             export EXEMPT_FROM_THROTTLE="127.0.0.1,192.168.0.2,::1"
             export APP_NAME=local
             export APP_URL=http://localhost:3000
             export DB_HOST=localhost
             export DB_PORT=5432
             export DB_CONNECTION=pgsql
             export TIMEZONE=Africa/Lagos
             export SSLMODE=disable
             export USERNAME=admin
             export PASSWORD=securepassword
             export DB_NAME=hng_db
             export MIGRATE=false
             export TEST_DB_HOST=localhost
             export TEST_DB_PORT=5432
             export TEST_DB_CONNECTION=pgsql
             export TEST_TIMEZONE=Africa/Lagos
             export TEST_SSLMODE=disable
             export TEST_USERNAME=admin
             export TEST_PASSWORD="securepassword"
             export TEST_DB_NAME=hng_db
             export TEST_MIGRATE=true
             export IPSTACK_KEY=your_ipstack_key
             export IPSTACK_BASE_URL=http://api.ipstack.com
           dest: /opt/stage_5b/.env
           owner: hng
           group: hng
           mode: '0644'

       - name: Copy and update run_production_app.sh
         copy:
           content: |
             #!/bin/bash
             cd /opt/stage_5b
             source .env
             ./hng_boilerplate_golang_web
           dest: /opt/stage_5b/run_production_app.sh
           owner: hng
           group: hng
           mode: '0755'

       - name: Set up logging directory
         file:
           path: /var/log/stage_5b
           state: directory
           owner: hng
           group: hng
           mode: '0755'

       - name: Configure error log
         copy:
           content: ""
           dest: /var/log/stage_5b/error.log
           owner: hng
           group: hng
           mode: '0644'

       - name: Configure stdout log
         copy:
           content: ""
           dest: /var/log/stage_5b/out.log
           owner: hng
           group: hng
           mode: '0644'

       - name: Build the application as hng user
         command: >
           sudo -u hng bash -c '
           cd /opt/stage_5b &&
           go build -buildvcs=false > /var/log/stage_5b/build.log 2>&1'
         register: app_build
         ignore_errors: yes

       - name: Display build log
         shell: tail -n 50 /var/log/stage_5b/build.log
         register: build_log

       - name: Display build log output
         debug:
           msg: "{{ build_log.stdout }}"

       - name: Run the application as hng user
         command: >
           sudo -u hng bash -c '
           cd /opt/stage_5b &&
           source .env &&
           ./run_production_app.sh > /var/log/stage_5b/out.log 2> /var/log/stage_5b/error.log &'
         register: app_run
         ignore_errors: yes

       - name: Display application logs
         shell: tail -n 50 /var/log/stage_5b/out.log
         register: app_log

       - name: Display application log output
         debug:
           msg: "{{ app_log.stdout }}"

       - name: Display error logs
         shell: tail -n 50 /var/log/stage_5b/error.log
         register: error_log

       - name: Display error log output
         debug:
           msg: "{{ error_log.stdout }}"

       - name: Check if the application is running
         shell: |
           pgrep -fl hng_boilerplate_golang_web
         register: app_running

       - name: Display application status
         debug:
           msg: "Application running status: {{ app_running.stdout }}"

       - name: Configure Nginx
         template:
           src: nginx.conf.j2
           dest: /etc/nginx/sites-available/default
         notify:
           - restart nginx

     handlers:
       - name: restart nginx
         service:
           name: nginx
           state: restarted
   ```

3. **Create the Nginx Template:**

   `nginx.conf.j2`
   ```nginx
   server {
       listen 80;

       server_name _;

       location / {
           proxy_pass http://127.0.0.1:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       error_log /var/log/stage_5b/error.log;
       access_log /var/log/stage_5b/out.log;
   }
   ```

#### Step 3: Run the Ansible Playbook

1. **Run the playbook:**

   ```bash
   ansible-playbook main.yaml -i inventory.cfg -b
   ```

#### Step 4: Verify the Deployment

1. **Check Application Status:**
   SSH into your EC2 instance and check if the

 application is running on port 3000.

   ```bash
   ssh -i /path/to/your-key-pair.pem ubuntu@your-ec2-public-ip
   sudo ss -tulnp | grep :3000
   ```

2. **Check Log Files:**
   Ensure the application started correctly by checking the log files.

   ```bash
   tail -n 50 /var/log/stage_5b/out.log
   tail -n 50 /var/log/stage_5b/error.log
   tail -n 50 /var/log/stage_5b/build.log
   ```

3. **Verify Nginx Configuration:**
   Ensure Nginx is configured correctly and running.

   ```bash
   sudo nginx -t
   sudo systemctl status nginx
   ```

4. **Access the Application:**
   Open a web browser and navigate to `http://your-ec2-public-ip`.

### Troubleshooting

1. **Application Not Running:**
   - **Check Application Status:**
     ```bash
     sudo ss -tulnp | grep :3000
     ```
   - **Check Log Files:**
     ```bash
     tail -n 50 /var/log/stage_5b/out.log
     tail -n 50 /var/log/stage_5b/error.log
     tail -n 50 /var/log/stage_5b/build.log
     ```
   - **Run the Application Manually:**
     ```bash
     sudo -u hng -i
     cd /opt/stage_5b
     source .env
     ./hng_boilerplate_golang_web
     ```

2. **502 Bad Gateway Error:**
   - **Verify Application Running on Port 3000:**
     Ensure the application is running and accessible on port 3000 internally.
     ```bash
     sudo ss -tulnp | grep :3000
     ```
   - **Check Nginx Configuration:**
     Ensure Nginx is configured correctly and pointing to the correct internal address and port.
     ```bash
     sudo nginx -t
     sudo systemctl status nginx
     ```
   - **Check Nginx Logs:**
     ```bash
     tail -n 50 /var/log/stage_5b/error.log
     tail -n 50 /var/log/nginx/error.log
     ```

3. **General Issues:**
   - **Review Environment Variables:**
     Ensure all required environment variables are set and correctly sourced.
     ```bash
     cat /opt/stage_5b/.env
     ```
   - **Check File Permissions:**
     Ensure the `hng` user has the necessary permissions to read and execute files.
     ```bash
     sudo chown -R hng:hng /opt/stage_5b
     sudo chmod -R 755 /opt/stage_5b
     ```

### Verify Deployment

1. **Run the Updated Playbook:**
   ```bash
   ansible-playbook main.yaml -i inventory.cfg -b
   ```

2. **Check Application Status and Logs:**
   ```bash
   ssh -i /path/to/your-key-pair.pem ubuntu@your-ec2-public-ip
   sudo ss -tulnp | grep :3000
   tail -n 50 /var/log/stage_5b/out.log
   tail -n 50 /var/log/stage_5b/error.log
   tail -n 50 /var/log/stage_5b/build.log
   ```

3. **Verify Nginx Configuration:**
   ```bash
   sudo nginx -t
   sudo systemctl status nginx
   ```

4. **Access the Application:**
   Open a web browser and navigate to `http://your-ec2-public-ip`.
```

This README is compatible with GitHub and provides clear and detailed steps for deploying the application using Ansible. It includes instructions for setting up the EC2 instance, preparing the Ansible playbook, running the playbook, and verifying the deployment. Additionally, it provides troubleshooting steps for common issues.
