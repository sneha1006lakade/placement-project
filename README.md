# $${\color{red} \textbf{Project}: \textbf{Angular}  \ \textbf{App}}$$

### Tech Stack
- Frontend: Angular,Nodejs
- Backend: Java-Springboot
- Database: Mariadb
- Cloud:AWS:ec2,s3,cloudfront,rds,route-53,certificate manager
### Summary:
- Launch 3 ec2 instances for Frontend, Backend, and Database respectively
- step1: Set Up Database Instance
- step2: Set Up Backend
- step3: Set UP Frontend
  
## $${\color{blue} \textbf{Set} \textbf{Up}  \ \textbf{Database}}$$

### Installing MariaDB, Setting Password, and Importing Database on Ubuntu Linux


###  ${\color{blue} \textbf{Create } \textbf{RDS}  \ \textbf{Instance}}$

![db](https://github.com/abhipraydhoble/Project-Angular-App/assets/122669982/8d992b33-4a08-4a68-95ab-1021c1111791)


```bash
sudo apt update
sudo apt install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb


```
### ${\color{green} \textbf{Login } \textbf{Into}  \ \textbf{Database}}$
````
sudo mysql -h database-1.cxqukacgq5pj.us-east-1.rds.amazonaws.com -u Angular-db -pPasswd123$
````
```sql
CREATE DATABASE springbackend;

```

### ${\color{green} \textbf{Import} \textbf{ Database}  \ \textbf{From} \textbf{ SQL} \textbf{ File}}$
```bash
sudo mysql -h database-1.cxqukacgq5pj.us-east-1.rds.amazonaws.com -u Angular-db -pPasswd123$ springbackend < springbackend.sql
```
```bash
sudo mysql -h database-1.cxqukacgq5pj.us-east-1.rds.amazonaws.com -u Angular-db -pPasswd123$
```
```sql
show databases;
show tables;
select * from tbl_workers;
```
