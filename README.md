
# Hosting a Flask Login Page on an Ubuntu EC2 Instance with AWS RDS MySQL

## **Step 1: Set Up the Ubuntu EC2 Instance**

### 1. Launch EC2 Instance
- Choose **Ubuntu Server 22.04 LTS** and `t2.micro` as the instance type.
- Set up **Security Group**: allow SSH (port 22) and HTTP (port 80).
- Key Pair: Create or use an existing one.

### 2. Connect to EC2
- Use SSH to connect to your instance:
  ```bash
  ssh -i "your-key.pem" ubuntu@<EC2-Public-IP>
  ```

### 3. Install Required Software
- Update the package list and install necessary software:
  ```bash
  sudo apt update
  sudo apt install nginx mysql-client python3 python3-pip python3-venv -y
  ```

### 4. Set Up Python Virtual Environment
- Create a project directory, set up a Python virtual environment, and install dependencies:
  ```bash
  mkdir ~/login_app
  cd ~/login_app
  python3 -m venv myappenv
  source myappenv/bin/activate
  pip install flask pymysql
  ```

---

## **Step 2: Set Up AWS RDS MySQL Database**

### 1. Create an RDS Instance
- Go to the **AWS RDS Console**, select **MySQL**, and configure the instance.
- Enable **7-day backup retention** and **point-in-time recovery**.

### 2. Allow RDS Connections
- Configure the RDS security group to allow incoming traffic on port **3306** (MySQL).
- Note the **RDS Endpoint** for connecting to the database from EC2.

---

## **Step 3: Create the Flask Login Application**

### 1. Create Flask Application
- Inside your project folder (`~/login_app`), create `app.py`:
  ```bash
  nano app.py
  ```
  Example `app.py`:
  ```python
  from flask import Flask, render_template, request
  import pymysql

  app = Flask(__name__)

  # Connect to AWS RDS MySQL
  db = pymysql.connect(
      host="your-rds-endpoint",
      user="admin",
      password="YourStrongPassword",
      database="login_db"
  )

  @app.route('/')
  def index():
      return render_template('login.html')

  @app.route('/login', methods=['POST'])
  def login():
      cursor = db.cursor()
      cursor.execute("SELECT * FROM users WHERE username=%s AND password=%s", 
                     (request.form['username'], request.form['password']))
      result = cursor.fetchone()
      return "Login Successful" if result else "Login Failed"

  if __name__ == '__main__':
      app.run(host='0.0.0.0', port=5000)
  ```

### 2. Create Login HTML Page
- Create a `templates` folder and add the login form:
  ```bash
  mkdir templates
  nano templates/login.html
  ```
  Example `login.html`:
  ```html
  <form action="/login" method="post">
      <label>Username:</label><input type="text" name="username"><br>
      <label>Password:</label><input type="password" name="password"><br>
      <input type="submit" value="Login">
  </form>
  ```

### 3. Set Up MySQL Database
- Use the MySQL client to connect to your RDS instance and create the `users` table:
  ```bash
  mysql -h your-rds-endpoint -u admin -p
  ```
  MySQL commands:
  ```sql
  CREATE DATABASE login_db;
  USE login_db;
  CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(50));
  INSERT INTO users (username, password) VALUES ('testuser', 'testpassword');
  ```

### 4. Run the Flask Application
- Run your Flask application:
  ```bash
  python app.py
  ```

---

## **Step 4: Configure Nginx to Serve the Flask Application**

### 1. Set Up Nginx Proxy
- Create a new Nginx config file:
  ```bash
  sudo nano /etc/nginx/sites-available/login_app
  ```
- Add the following Nginx configuration:
  ```nginx
  server {
      listen 80;
      server_name your-ec2-public-ip;

      location / {
          proxy_pass http://127.0.0.1:5000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

### 2. Enable the Site and Restart Nginx
- Enable the site and restart Nginx:
  ```bash
  sudo ln -s /etc/nginx/sites-available/login_app /etc/nginx/sites-enabled
  sudo systemctl restart nginx
  ```

---

## **Step 5: Test the Application**

- Open your browser and navigate to `http://<EC2-Public-IP>`.
- Enter the credentials you added to the `users` table (e.g., `testuser` and `testpassword`).
- Verify that you are able to log in successfully.

---

This completes the setup of a Flask-based login page hosted on an Ubuntu EC2 instance, connected to an AWS RDS MySQL database, and served via Nginx. Let me know if you need further clarification or adjustments!
