# $${\color{red} \textbf{Project}: \textbf{Student}  \ \textbf{form} \ \textbf{on Terraform}}$$


**create profile with access key and secret key**
````
aws configure --profile <profile name>
````
**add provider**
````
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.54.1"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  profile = "profile name"

}
````
````
terraform init
````
**create a vpc**
````
resource "aws_vpc" "vpc" {
    cidr_block = "10.0.0.0/16"

    tags = {
        Name = "aws_vpc",
    } 
}
````
**create a public subnet**
````
resource "aws_subnet" "nginx-subnet" {
  vpc_id = aws_vpc.vpc.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true 

  tags = {
    Name = "Public-subnet_1",
  }
}
````
**create a private subnet for tomcat server**
````
resource "aws_subnet" "tom-subnet" {
  vpc_id = aws_vpc.vpc.id
  cidr_block = "10.0.20.0/25"
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = false 

  tags = {
    Name = "Private-subnet-1"
  }

}
````
**create a private subnet for database**
````
resource "aws_subnet" "db-subnet" {
  vpc_id = aws_vpc.vpc.id
  cidr_block = "10.0.30.0/26"
  availability_zone = "us-east-1c"
  map_public_ip_on_launch = false 

  tags = {
    Name = "Private-subnet-2"
  }
}
````
**create a internet Gateway**
````
resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.vpc.id
    tags = {
      Name = "aws_internet_gateway_1"
    }
  
}
````
**create elastic ip address**
````
resource "aws_eip" "elasticip" {
 domain = "vpc"
}
````
**create a Nat gateway**
````
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.elasticip.id
  subnet_id = aws_subnet.nginx-subnet.id
}
````
**create a public route table**
````
resource "aws_route_table" "RT1" {
    vpc_id = aws_vpc.vpc.id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.igw.id
    
    }

}
````
**add public subnet public route association**
````
resource "aws_route_table_association" "association" {
  subnet_id = aws_subnet.nginx-subnet.id
  route_table_id = aws_route_table.RT1.id
}
````
**create a private route table**
````
resource "aws_route_table" "RT2" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat.id
  }
}
````
**add both private subnet private route association**
````
resource "aws_route_table_association" "private-association-table" {
  subnet_id = aws_subnet.tom-subnet.id
  route_table_id = aws_route_table.RT2.id
}

resource "aws_route_table_association" "private-association-table-2" {
  subnet_id = aws_subnet.db-subnet.id
  route_table_id = aws_route_table.RT2.id
}
````
**create a security group**
````
resource "aws_security_group" "sg1" {
  name = "Three-tier"
  description = "Allow SSH,mysql,tomcat access"
  vpc_id = aws_vpc.vpc.id
}
````
**Add port 22,80,8080,3306 in inbound rule**
````
resource "aws_vpc_security_group_ingress_rule" "ssh" {
  security_group_id = aws_security_group.sg1.id
  cidr_ipv4 = "0.0.0.0/0"
  from_port = 22
  ip_protocol = "tcp"
  to_port = 22
}

resource "aws_vpc_security_group_ingress_rule" "tomcat" {
  security_group_id = aws_security_group.sg1.id
  cidr_ipv4 = "0.0.0.0/0"
  from_port = 8080
  ip_protocol = "tcp"
  to_port = 8080
}

resource "aws_vpc_security_group_ingress_rule" "nginx" {
  security_group_id = aws_security_group.sg1.id
  cidr_ipv4 = "0.0.0.0/0"
  from_port = 80
  ip_protocol = "tcp"
  to_port = 80
}

resource "aws_vpc_security_group_ingress_rule" "mysql" {
  security_group_id = aws_security_group.sg1.id
  cidr_ipv4 = "0.0.0.0/0"
  from_port = 3306
  ip_protocol = "tcp"
  to_port = 3306
  
}
````
**add outbount rule**
````
resource "aws_vpc_security_group_egress_rule" "outbound" {
  security_group_id = aws_security_group.sg1.id
  cidr_ipv4 = "0.0.0.0/0"
  ip_protocol = "-1"
}
````
**create a key pair from GUI and use that key name when we creat a instance**

**Add network interface with public subnet to connet to public instance /nginx instance**
````
resource "aws_network_interface" "network1" {
  subnet_id = aws_subnet.nginx-subnet.id
  private_ip = "10.0.1.100/24"
}
````
**Create a instance for NGINX**
````
resource "aws_instance" "nginx" {
  ami = "ami-08a0d1e16fc3f61ea"
  instance_type = "t2.micro"
  key_name = "terraform-key"
   network_interface {
    network_interface_id = aws_network_interface.network1.id
    device_index = 0
  }
   tags = {
    Name = "Nginx"
  }
}
````
**Attach the security group to instance**
````
resource "aws_network_interface_sg_attachment" "sg2" {
  security_group_id = aws_security_group.sg1.id
  network_interface_id = aws_instance.nginx.primary_network_interface_id
}
````
**Add network interface with private subnet 1 to connet to tomcat instance**
````
resource "aws_network_interface" "private-network1" {
  subnet_id = aws_subnet.tom-subnet.id
  private_ip = "10.0.20.100/24"
}
````
**Create a instance for tomcat**
````
resource "aws_instance" "tomcat" {
  ami = "ami-08a0d1e16fc3f61ea"
  instance_type = "t2.micro"
  key_name = "terraform-key"
  network_interface {
    network_interface_id = aws_network_interface.private-network1.id
    device_index = 0
}
  tags = {
    Name = "Tomcat"
  }
}
````
**Attach the security group to instance** 
````
resource "aws_network_interface_sg_attachment""sg-private"{
  security_group_id = aws_security_group.sg1.id
  network_interface_id = aws_instance.tomcat.primary_network_interface_id
}
````
**Add network interface with private subnet to connet to Database instance**
````
resource "aws_network_interface" "private-network2" {
  subnet_id = aws_subnet.db-subnet.id
  private_ip = "10.0.30.100"
}
````
**Create a instance for Database**
````
resource "aws_instance" "database" {
  ami = "ami-08a0d1e16fc3f61ea"
  instance_type = "t2.micro"
  key_name = "terraform-key"
  network_interface {
    network_interface_id = aws_network_interface.private-network2.id
    device_index = 0
  }
  tags = {
    Name = "Database"
  }
  }
````
**Attach the security group to instance**
````
resource "aws_network_interface_sg_attachment" "sg-private-2" {
  security_group_id = aws_security_group.sg1.id
  network_interface_id = aws_instance.database.primary_network_interface_id
}
````
**Create a subnet group for Database**
````
resource "aws_db_subnet_group" "db-subnet-group" {
  name = "db-subnet-group"
  subnet_ids = [aws_subnet.tom-subnet.id, aws_subnet.db-subnet.id]
}
````
**Create a Database with created VPC**
````
resource "aws_db_instance" "rds" {
  allocated_storage = 20
  db_name = "student"
  engine = "mariadb"
  engine_version = "10.11.6"
  username = "admin"
  password = "passwd123"
  instance_class = "db.t3.micro"
  skip_final_snapshot = true
  db_subnet_group_name = aws_db_subnet_group.db-subnet-group.name

  vpc_security_group_ids = [aws_security_group.sg1.id]

  tags = {
    Name = "RDS"
  }
}
````
````
terraform plan
````
````
terraform apply -auto-approve
````
**Create a file with the name of terraform-key.pem and add a key content which key we assigned with instances**
````
vim terraform-key.pem
````
**give 600 permission to file**
````
chmod 600 terraform-key.pem
````
## ${\color{red} \textbf{Connect to Nginx Server}}$
**get access of nginx instance on vs code (ubuntu) with instance ssh link**
````
sudo ssh -i "terraform-key.pem" ec2-user@<public_ip>
````
### ${\color{blue} \textbf{Setup Nginx server}}$
````
hostname nginx
bash
````
````
yum install nginx -y
systemctl start nginx
````
````
vim terraform-key.pem
````
````
chmod 600 terraform-key.pem
````
## ${\color{red} \textbf{Connect to database server and setup}}$
````
ssh -i terraform-key.pem ec2-user@<pri_ip_db_server>
````
````
hostname database
bash
````
**install mariadb and start**
````
yum install mariadb105-server -y
systemctl start mariadb.server
````
### ${\color{green} \textbf{log in database}}$

````
mysql -h <endpoint of database> -u <uname> -p<password>
````
````
show database;
````
````
create database studentapp;
````
````
use studentapp;
````
### ${\color{blue} \textbf{Create Table in DB}}$
````
 CREATE TABLE if not exists students(student_id INT NOT NULL AUTO_INCREMENT,  
	student_name VARCHAR(100) NOT NULL,  
	student_addr VARCHAR(100) NOT NULL,   
	student_age VARCHAR(3) NOT NULL,      
	student_qual VARCHAR(20) NOT NULL,     
	student_percent VARCHAR(10) NOT NULL,   
	student_year_passed VARCHAR(10) NOT NULL,  
	PRIMARY KEY (student_id)  
);
````
````
show tables;
````
````
describe students;
````
````
exit
````
### ${\color{green} \textbf{log out database instance}}$
## ${\color{red} \textbf{Connect to Tomcat server and setup}}$
````
ssh -i terraform-key.pem ec2-user@<pri_ip_tom_server>
````
````
hostname tomcat
bash
````
````
yum install java -y
````
````
mkdir /opt/tomcat
````
**install apache tomcat binary package and extract it**
````
curl –O https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.89/bin/apache-tomcat-9.0.89.tar.gz
````
````
tar -xzvf apache-tomcat-9.0.89.tar.gz -C /opt/tomcat/
````
````
cd /opt/tomcat/apache-tomcat-9.0.89/webapps/
````
**install student war file**
````
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war
````
````
cd /opt/tomcat/apache-tomcat-9.0.89/lib/
````
````
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar
````
### ${\color{blue} \textbf{Modify Apache Tomcat/context.xml}}$
````
cd /opt/tomcat/apache-tomcat-9.0.89/conf/
vim context.xml
````
**add username ,password ,DB-Endpoint ,DB-name**
**add it under the context wor at the line of 21**
````
 <Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
               maxTotal="100" maxIdle="30" maxWaitMillis="10000"
               username="USERNAME" password="PASSWORD" driverClassName="com.mysql.jdbc.Driver"
               url="jdbc:mysql://DB-ENDPOINT:3306/DATABASE"/>
````
**start Tomcat**
````
cd bin
chmod +x catalina.sh
./catalina.sh start
````
## ${\color{red} \textbf{Come back to Nginx server}}$
### ${\color{green} \textbf{set up proxy in nginx.conf}}$
````
vim /etc/nginx/nginx.conf
````
**add proxy in nginx.conf file at the line pf 47 under error_page**
````
:set nu
````
````
location / {
proxy_pass http://private-IP-tomcat:8080/student/;
}
````
````
systemctl restart nginx
````
### $\color{red} \textbf{Go \ To \ Browser \ Hit \ Public-IP \ Nginx}$
![image](https://github.com/aarti215/student-app-3-tier-project/assets/171672859/cda6b022-9003-438b-89d4-fc00a3fe4e3a)



### $\color{red} \textbf{Here \ are \ the \ Output \ -Registered \ Student-Data}$

![image](https://github.com/aarti215/student-app-3-tier-project/assets/171672859/c11d08d4-a5fa-4a11-8816-0a5b9ffd2b65)








  
