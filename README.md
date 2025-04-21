# pythonwithfastapi_deploy

Here’s a **FastAPI + PostgreSQL CRUD** example with a clean folder structure:

---

### 🗂 Folder Structure

```
fastapi_crud/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models/
│   │   └── user.py
│   ├── schemas/
│   │   └── user.py
│   ├── crud/
│   │   └── user.py
│   ├── db/
│   │   ├── base.py
│   │   └── session.py
│   └── api/
│       └── routes/
│           └── user.py
├── alembic.ini
├── requirements.txt
```

---

### 📦 `requirements.txt`

```txt
fastapi
uvicorn
sqlalchemy
psycopg2-binary
pydantic
alembic
```

---

### ⚙️ `db/session.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://username:password@localhost/dbname"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
```

---

### ⚙️ `db/base.py`

```python
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base()
```

---

### 🧩 `models/user.py`

```python
from sqlalchemy import Column, Integer, String
from app.db.base import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

---

### 📐 `schemas/user.py`

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

class UserRead(UserCreate):
    id: int

    class Config:
        orm_mode = True
```

---

### ⚙️ `crud/user.py`

```python
from sqlalchemy.orm import Session
from app.models.user import User
from app.schemas.user import UserCreate

def create_user(db: Session, user: UserCreate):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def get_users(db: Session):
    return db.query(User).all()

def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

def delete_user(db: Session, user_id: int):
    user = db.query(User).get(user_id)
    db.delete(user)
    db.commit()
```

---

### 🔁 `api/routes/user.py`

```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from app.db.session import SessionLocal
from app.schemas.user import UserCreate, UserRead
from app.crud import user as crud_user

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/users", response_model=UserRead)
def create(user: UserCreate, db: Session = Depends(get_db)):
    return crud_user.create_user(db, user)

@router.get("/users", response_model=list[UserRead])
def read_users(db: Session = Depends(get_db)):
    return crud_user.get_users(db)
```

---

### 🚀 `main.py`

```python
from fastapi import FastAPI
from app.api.routes import user
from app.db.base import Base
from app.db.session import engine

Base.metadata.create_all(bind=engine)

app = FastAPI()
app.include_router(user.router)
```

---

Here’s how to add **Docker** support to the FastAPI + PostgreSQL CRUD project:

---

### 🐳 `Dockerfile`

```Dockerfile
# Use official Python image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Copy dependencies
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY ./app ./app

# Expose port
EXPOSE 8000

# Run FastAPI app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

---

### 🐳 `docker-compose.yml`

```yaml
version: '3.9'

services:
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: fastapidb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  web:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/fastapidb

volumes:
  postgres_data:
```

---

### 🔧 Update `db/session.py` for Docker

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://postgres:postgres@localhost/fastapidb")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
```

---

### ✅ Run Project

```bash
docker-compose up --build
```

---

Here’s a minimal **Terraform setup** to deploy a **FastAPI + PostgreSQL** app on **AWS** using **EC2 + RDS + S3**:

---

### ✅ 1. **Folder Structure**
```
terraform/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── ec2.tf
├── rds.tf
├── s3.tf
```

---

### ✅ 2. **main.tf**
```hcl
provider "aws" {
  region = var.region
}
```

---

### ✅ 3. **variables.tf**
```hcl
variable "region" {
  default = "ap-south-1"
}

variable "db_username" {}
variable "db_password" {}
```

---

### ✅ 4. **ec2.tf**
```hcl
resource "aws_instance" "fastapi_server" {
  ami           = "ami-0fc5d935ebf8bc3bc" # Ubuntu (Mumbai)
  instance_type = "t2.micro"
  key_name      = "your-key"

  tags = {
    Name = "FastAPI Server"
  }

  user_data = <<-EOF
              #!/bin/bash
              apt update
              apt install docker.io -y
              apt install git -y
              git clone https://github.com/your/repo.git
              cd repo
              docker-compose -f docker-compose.prod.yml up -d
              EOF
}
```

---

### ✅ 5. **rds.tf**
```hcl
resource "aws_db_instance" "postgres" {
  allocated_storage    = 20
  engine               = "postgres"
  instance_class       = "db.t3.micro"
  name                 = "fastapidb"
  username             = var.db_username
  password             = var.db_password
  publicly_accessible  = true
  skip_final_snapshot  = true
}
```

---

### ✅ 6. **s3.tf**
```hcl
resource "aws_s3_bucket" "fastapi_bucket" {
  bucket = "fastapi-app-files-bucket"
  acl    = "private"
}
```

---

### ✅ 7. **outputs.tf**
```hcl
output "ec2_public_ip" {
  value = aws_instance.fastapi_server.public_ip
}

output "rds_endpoint" {
  value = aws_db_instance.postgres.endpoint
}
```

---

### ✅ 8. Initialize + Apply

```bash
terraform init
terraform apply -var="db_username=admin" -var="db_password=yourpass123"
```

---

Great! Here’s how to deploy your **FastAPI app with PostgreSQL** on **AWS ECS (Fargate)** using **Terraform**, without managing EC2 instances directly.

---

### ✅ 1. **Folder Structure**
```
terraform/
│
├── main.tf
├── variables.tf
├── ecs.tf
├── rds.tf
├── iam.tf
├── s3.tf
├── outputs.tf
```

---

### ✅ 2. **main.tf**
```hcl
provider "aws" {
  region = var.region
}
```

---

### ✅ 3. **variables.tf**
```hcl
variable "region"         { default = "ap-south-1" }
variable "app_name"       { default = "fastapi-app" }
variable "db_username"    {}
variable "db_password"    {}
variable "docker_image"   {}  # ECR or Docker Hub
```

---

### ✅ 4. **ecs.tf**
```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-cluster"
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.app_name}-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([{
    name  = "fastapi"
    image = var.docker_image
    portMappings = [{ containerPort = 80, hostPort = 80 }]
    environment = [
      { name = "DB_HOST", value = aws_db_instance.postgres.address },
      { name = "DB_USER", value = var.db_username },
      { name = "DB_PASS", value = var.db_password }
    ]
  }])

  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn
}

resource "aws_ecs_service" "app" {
  name            = "${var.app_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  launch_type     = "FARGATE"
  desired_count   = 1

  network_configuration {
    subnets          = ["subnet-xxxxxx"] # update with actual subnet IDs
    assign_public_ip = true
    security_groups  = ["sg-xxxxxx"]     # update with actual SG
  }
}
```

---

### ✅ 5. **rds.tf**
```hcl
resource "aws_db_instance" "postgres" {
  engine            = "postgres"
  instance_class    = "db.t3.micro"
  name              = "fastapidb"
  username          = var.db_username
  password          = var.db_password
  allocated_storage = 20
  skip_final_snapshot = true
  publicly_accessible = true
}
```

---

### ✅ 6. **iam.tf**
```hcl
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

---

### ✅ 7. **s3.tf (Optional)**
```hcl
resource "aws_s3_bucket" "fastapi_bucket" {
  bucket = "${var.app_name}-bucket"
  acl    = "private"
}
```

---

### ✅ 8. **outputs.tf**
```hcl
output "rds_endpoint" {
  value = aws_db_instance.postgres.endpoint
}
```

---

### ✅ 9. Deploy

```bash
terraform init
terraform apply -var="db_username=admin" -var="db_password=secret123" -var="docker_image=yourdocker/image"
```

---
Here's how to add an **Application Load Balancer (ALB)** for your FastAPI app deployed on **AWS ECS Fargate** using Terraform:

---

### ✅ **Update Folder Structure**
Add this file:

```
terraform/
├── alb.tf
```

---

### ✅ **alb.tf**
```hcl
resource "aws_lb" "fastapi_alb" {
  name               = "${var.app_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = ["sg-xxxxxx"] # public SG
  subnets            = ["subnet-xxxx", "subnet-yyyy"] # public subnets
}

resource "aws_lb_target_group" "fastapi_tg" {
  name     = "${var.app_name}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = "vpc-xxxxxx" # your VPC ID

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200-399"
  }
}

resource "aws_lb_listener" "fastapi_listener" {
  load_balancer_arn = aws_lb.fastapi_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
  }
}
```

---

### ✅ **Update ECS Service in `ecs.tf`**
```hcl
resource "aws_ecs_service" "app" {
  ...

  load_balancer {
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
    container_name   = "fastapi"
    container_port   = 80
  }

  depends_on = [aws_lb_listener.fastapi_listener]
}
```

---

### ✅ **Final Output (Optional in outputs.tf)**
```hcl
output "alb_dns" {
  value = aws_lb.fastapi_alb.dns_name
}
```

---

To attach a **custom domain** with **HTTPS** to your **FastAPI app on AWS ECS Fargate**, follow this using **Terraform**, **Route 53**, and **ACM (SSL)**:

---

### ✅ **1. Get/Verify Domain in Route 53**
If your domain (e.g., `api.example.com`) is already in Route 53:
- Note the Hosted Zone ID.

---

### ✅ **2. Request SSL Certificate – `acm.tf`**
```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = "api.example.com"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  name    = aws_acm_certificate.cert.domain_validation_options[0].resource_record_name
  type    = aws_acm_certificate.cert.domain_validation_options[0].resource_record_type
  zone_id = "Z1234567890" # your Route 53 Hosted Zone ID
  records = [aws_acm_certificate.cert.domain_validation_options[0].resource_record_value]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "cert_validation" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [aws_route53_record.cert_validation.fqdn]
}
```

---

### ✅ **3. Update ALB Listener to Use HTTPS – `alb.tf`**
```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.fastapi_alb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = aws_acm_certificate.cert.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
  }

  depends_on = [aws_acm_certificate_validation.cert_validation]
}
```

---

### ✅ **4. Add DNS Record for Domain – `route53.tf`**
```hcl
resource "aws_route53_record" "app_domain" {
  zone_id = "Z1234567890" # your Hosted Zone ID
  name    = "api.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.fastapi_alb.dns_name
    zone_id                = aws_lb.fastapi_alb.zone_id
    evaluate_target_health = true
  }
}
```

---

Once applied, your FastAPI app will be live at:

```
https://api.example.com
```

To enable **SSL (HTTPS)** for your FastAPI app on **AWS ECS Fargate**, you need to:

1. **Obtain an SSL certificate** using **AWS ACM (AWS Certificate Manager)**.
2. **Configure your Application Load Balancer (ALB)** to serve traffic over **HTTPS**.
3. **Update DNS records** using **Route 53** for the custom domain to point to your ALB.

Here's how you can achieve this in **Terraform**.

---

### ✅ **1. Request SSL Certificate (ACM)**

First, create an SSL certificate with **AWS ACM**. This will validate your domain using **DNS**.

**Create `acm.tf`:**
```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = "api.example.com"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  name    = aws_acm_certificate.cert.domain_validation_options[0].resource_record_name
  type    = aws_acm_certificate.cert.domain_validation_options[0].resource_record_type
  zone_id = "Z1234567890"  # Your Route 53 Hosted Zone ID
  records = [aws_acm_certificate.cert.domain_validation_options[0].resource_record_value]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "cert_validation" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [aws_route53_record.cert_validation.fqdn]
}
```

---

### ✅ **2. Update ALB to Handle HTTPS**

Update the **Application Load Balancer (ALB)** to listen on **HTTPS (port 443)** and use the ACM certificate.

**Create `alb.tf`:**
```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.fastapi_alb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"  # Standard SSL policy
  certificate_arn   = aws_acm_certificate.cert.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
  }

  depends_on = [aws_acm_certificate_validation.cert_validation]
}
```

---

### ✅ **3. Create DNS Record for Your Domain**

Use **Route 53** to set up a DNS record pointing to your ALB, so your domain resolves to your application.

**Create `route53.tf`:**
```hcl
resource "aws_route53_record" "app_domain" {
  zone_id = "Z1234567890"  # Your Route 53 Hosted Zone ID
  name    = "api.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.fastapi_alb.dns_name
    zone_id                = aws_lb.fastapi_alb.zone_id
    evaluate_target_health = true
  }
}
```

---

### ✅ **4. Optional: HTTP to HTTPS Redirection**

If you want to ensure that all traffic is redirected to HTTPS, you can set up an HTTP listener on port 80 to automatically redirect to HTTPS.

**Add this to `alb.tf` for HTTP → HTTPS redirection:**
```hcl
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.fastapi_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "redirect"
    target_group_arn = aws_lb_target_group.fastapi_tg.arn
    redirect {
      protocol = "HTTPS"
      port     = "443"
      status_code = "HTTP_301"
    }
  }
}
```

---

### ✅ **Final Steps: Apply Terraform Configuration**

1. **Initialize Terraform**:
   ```bash
   terraform init
   ```

2. **Apply the Configuration**:
   ```bash
   terraform apply
   ```

---

### Your FastAPI app will now be live at:
- `https://api.example.com`

---

