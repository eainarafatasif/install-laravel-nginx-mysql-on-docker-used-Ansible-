# install-laravel-nginx-mysql-on-docker-used-Ansible-
To install Laravel, Nginx, and MySQL on Docker using Ansible, you need to break it down into tasks for each component. Here’s an overview of how to set this up:

To install Laravel, Nginx, and MySQL on Docker using Ansible, you need to break it down into tasks for each component. Here’s an overview of how to set this up:

### Steps Overview:
1. **Install Docker**: Ensure Docker is installed on the target server.
2. **Install Docker Compose**: Use Docker Compose to define and manage multi-container applications (Laravel, Nginx, MySQL).
3. **Install Laravel**: Pull and set up a Laravel application.
4. **Configure Nginx**: Set up Nginx to serve the Laravel application.
5. **Configure MySQL**: Create a MySQL container with the necessary database and credentials for Laravel.
6. **Run Docker Compose**: Build and run the services using Docker Compose.

### Prerequisites:
- Ansible installed on your control machine.
- The target server accessible via SSH.
- Docker installed on the target server.
  
### Directory Structure:
```
ansible/
├── playbook.yml
├── roles/
    ├── docker/
    ├── laravel/
    ├── nginx/
    └── mysql/
```

### 1. **Create Ansible Playbook**
Create a playbook (`playbook.yml`) to orchestrate the process.

```yaml
---
- hosts: servers
  become: yes
  tasks:
    - include_role:
        name: docker
    
    - include_role:
        name: laravel

    - include_role:
        name: nginx

    - include_role:
        name: mysql
```

### 2. **Docker Role**
The Docker role will install Docker and Docker Compose on the server.

`roles/docker/tasks/main.yml`:
```yaml
---
- name: Install Docker
  apt:
    name: docker.io
    state: present
    update_cache: yes

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}"
    dest: /usr/local/bin/docker-compose
    mode: '0755'
  become: yes

- name: Start and enable Docker service
  service:
    name: docker
    state: started
    enabled: true
```

### 3. **Laravel Role**
This will pull the Laravel application from Git or set it up.

`roles/laravel/tasks/main.yml`:
```yaml
---
- name: Clone Laravel project from GitHub
  git:
    repo: "https://github.com/your-laravel-repo.git"
    dest: /var/www/laravel
    version: master

- name: Run Docker Compose to build Laravel environment
  shell: docker-compose up -d
  args:
    chdir: /var/www/laravel
```

**Note**: Update the repo URL to your Laravel project.

### 4. **Nginx Role**
Configure Nginx to serve the Laravel application.

`roles/nginx/tasks/main.yml`:
```yaml
---
- name: Create Nginx config for Laravel
  template:
    src: nginx.conf.j2
    dest: /var/www/laravel/nginx/nginx.conf

- name: Run Docker Compose to set up Nginx
  shell: docker-compose up -d nginx
  args:
    chdir: /var/www/laravel
```

`roles/nginx/templates/nginx.conf.j2`:
```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/laravel/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass   php-fpm:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

### 5. **MySQL Role**
Set up MySQL using Docker.

`roles/mysql/tasks/main.yml`:
```yaml
---
- name: Create MySQL Docker container
  docker_container:
    name: mysql
    image: mysql:5.7
    state: started
    restart_policy: always
    env:
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
      MYSQL_DATABASE: "{{ mysql_database }}"
      MYSQL_USER: "{{ mysql_user }}"
      MYSQL_PASSWORD: "{{ mysql_password }}"
    ports:
      - "3306:3306"
```

You should set MySQL variables in `group_vars/all.yml`:
```yaml
mysql_root_password: "your_root_password"
mysql_database: "laravel"
mysql_user: "laraveluser"
mysql_password: "your_password"
```

### 6. **Docker Compose File for Laravel**
Add a `docker-compose.yml` file in the Laravel directory to manage the containers for Laravel, Nginx, and MySQL.

```yaml
version: '3.7'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./public:/var/www/laravel/public
    depends_on:
      - php-fpm

  php-fpm:
    image: php:7.4-fpm
    container_name: php-fpm
    volumes:
      - .:/var/www/laravel
    working_dir: /var/www/laravel
    depends_on:
      - mysql

  mysql:
    image: mysql:5.7
    container_name: mysql
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_USER: laraveluser
      MYSQL_PASSWORD: your_password
      MYSQL_ROOT_PASSWORD: your_root_password
    ports:
      - "3306:3306"
```

### 7. **Run the Playbook**

Now, run the playbook to install Laravel, Nginx, and MySQL on Docker.

```bash
ansible-playbook -i inventory playbook.yml
```

### Inventory File:
Create an `inventory` file that lists your server(s):

`inventory`:
```
[servers]
your_server_ip ansible_user=your_username
```

This setup should configure and install Laravel, Nginx, and MySQL inside Docker containers using Ansible.
