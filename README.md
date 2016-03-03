# simple-ansible-aws-django
Scripts for setup Django on Ubuntu ec2 instance with Mysql RDS instance

Based on [how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-14-04)

### Notes

#### AWS
Main problem with Ansible and AWS is that Ansible modules uses different arguments and you should set many facts manually.
You can set up new private VPC as done here: [ansible-aws-vpc-ha-wordpress](https://github.com/arbabnazar/ansible-aws-vpc-ha-wordpress).
Or you should uses extra Ansible modules: [ec2_vpc_net_facts](https://docs.ansible.com/ansible/ec2_vpc_net_facts_module.html), [ec2_vpc_subnet_facts](https://docs.ansible.com/ansible/ec2_vpc_subnet_facts_module.html).

#### Django
Main problem with Django is that it keeps settings as python module that hard to modified with simple replace. Also Django cli does not read stdin input for many operations so you should create python scripts and pass it into Django shell. I recommended this Ansible role: [Stouts.django](https://galaxy.ansible.com/detail#/role/832).

#### Ansible
I used this core modules: [ec2](https://docs.ansible.com/ansible/ec2_module.html), [rds](https://docs.ansible.com/ansible/rds_module.html), [ec2_group](https://docs.ansible.com/ansible/ec2_group_module.html), [ec2_key](https://docs.ansible.com/ansible/ec2_key_module.html). Also [ec2_facts](https://docs.ansible.com/ansible/ec2_facts_module.html) may be useful. More info here: [list of all modules](https://docs.ansible.com/ansible/list_of_all_modules.html) and [ansible-quickref](https://github.com/lorin/ansible-quickref/blob/master/ec2.rst).

### Requirements

- Ansible
- boto
- AWS admin access

### AWS credentials
Ansible uses python-boto library to call AWS API, and boto needs AWS credentials in order to perform all the functions. There are many ways to configure your AWS credentials. The easiest way is to crate a .boto file under your user home directory:
```shell
$ cat ~/.boto
[Credentials]
aws_access_key_id = <your_access_key_here>
aws_secret_access_key = <your_secret_key_here>
```

### Configuration:
Edit the `all.yml` file inside the `group_vars` directory:
```yaml
# Tags
tags:
  Name: test-django
  ENV: test
  application: django
  server_role: webserver

# ec2_key
key_name: "devops-key"
ssh_key_location: ~/.ssh/id_rsa.pub 

# VPC
vpc_region: us-east-1
vpc_id: vpc-xxxx
vpc_subnet_id: subnet-yyyyy

# EC2
image: ami-ff427095 # ubuntu trusty 14.04

# RDS settings
rds_db_engine: MySQL
rds_db_size: 5
rds_db_name: django
rds_instance_type: db.t2.micro
rds_db_username: django
rds_db_password: changeThisToSecureOne!
backup_retention_period: 7

# Apps paths
projects_dir: /apps
```

### AWS Setup

```bash
ansible-playbook setup_aws.yml
```

`setup_aws.yml` create:
- create new AWS ssh key
- public Ubuntu Trusty Ec2 instance with public access
- private Mysql RDS instance with private access
- Security Group for Ec2 instance
- Security Group for RDS
- local file `rds_info.yml` with credentials for Mysql that already added into `.gitignore`

### Install Django

```bash
ansible-playbook -i ec2.py setup_django.yml
```

`setup_django.yml` install with apt:
- python-pip
- python-dev
- gcc
- git
- nginx
- gunicorn
- mysql-client
- python-mysqldb

And install with pip:
- django

And create:
- projects_dir

### Create Django project

```bash
ansible-playbook -i ec2.py create_project.yml
```

You should provide this variables to the `create_project.yml`:
- project_name
- project_dns
- project_admin_user
- project_admin_pass
- project_admin_mail

If you want download existed project, you should provide `git_url` variable. Project should have this structure:
manage.py
project_name/
..__init__.py
..settings.py
..urls.py
..wsgi.py

`create_project.yml` will:
- download or create project
- create `overrides` and `admin` files with auth information in project directory near `settings.py`
- override DATABASES and STATIC_ROOT settings
- execute Django db migration
- create superuser with given variables
- collect static files
- create Gunicorn config for Upstart personal for project that will start as `nobody:www-data`
- create Nginx config personal for project
- disable Nginx `default` config
- restart Nginx and Gunicorn
