# Containerization & Orchestartion Assignment


Architecture:
-------------

GitHub
                       |
                       | Push
                       v
                  +---------+
                  | Jenkins |
                  |   EC2   |
                  +----+----+
                       |
             Build + Test + Docker
                       |
              +--------+--------+
              |                 |
              v                 v
        ECR Frontend       ECR Backend
              |                 |
              +--------+--------+
                       |
                       v
                 Amazon EKS
                       |
              +--------+--------+
              |                 |
        Frontend Pods      Backend Pods
              |                 |
              +--------+--------+
                       |
                    Services
                       |
                  Ingress/ALB
                       |
                    Internet

                       |
                       v
              CloudWatch Logs
              Metrics / Alarms


--------------------------------------------------------------------------


What we will build

I recommend completing the assignment in this order:

Phase	What we'll implement
1	Fork repository
2	Understand MERN application
3	Create frontend Dockerfile
4	Create backend Dockerfile
5	Test both containers locally
6	Create ECR repositories
7	Push images to ECR
8	Configure AWS CLI
9	Install Jenkins on EC2
10	Configure Jenkins → GitHub
11	Configure Jenkins → ECR
12	Create EKS cluster
13	Create Kubernetes/Helm configuration
14	Deploy frontend/backend to EKS
15	Expose application through AWS Load Balancer
16	Configure CloudWatch
17	Configure centralized logging
18	Test scaling/failover
19	Document architecture
20	Optional SNS + Slack/Teams/Telegram