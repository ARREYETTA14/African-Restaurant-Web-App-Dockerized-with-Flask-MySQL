# African-Restaurant-Web-App-Dockerized-with-Flask-MySQL
Containerised African restaurant web app built with Flask and MySQL. Users select dishes, see images instantly, and place orders stored persistently in MySQL. Docker Compose orchestrates the multi-container setup for easy deployment and data management.

## Project Structure

```csharp
EC2 Instance:

restaurant-app/
│
├── docker-compose.yml
│
├── db/
│   └── init.sql
│
└── web/
    ├── Dockerfile
    ├── app.py
    ├── index.html
    ├── requirements.txt

S3 Bucket:
static/
    ├── styles.css
    └── images/
          ├── eru.jpg
          ├── egusi.jpg
          └── bunny.jpg
```

## Step 1: Create Your Project Folder

In you AWS Account, Launch an ```Amazon Linux 2023``` Ec2 instance with instance type at least ```t3.medium``` with ports, ```5000```.

SSH into the Instance:

- Install Docker and Docker Compose:
```bash
sudo dnf update -y
sudo dnf install docker -y
sudo service docker start
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```
Verify if Docker and Docker Compose are available:
```bash
docker version
docker compose version
```
Docker Compose v2 is now bundled into Docker in many recent distros, but just in case, here’s the manual method:

First, find out your system architecture
Run this:
```bash
uname -m
```
If you get:

```x86_64``` → you're on 64-bit Intel/AMD

```aarch64``` → you're on ARM (like Graviton)

Let’s assume x86_64 for now (most EC2 instances).

Run the Following Commands:

```bash
# Create the plugin directory
sudo mkdir -p /usr/local/lib/docker/cli-plugins

# Download Compose v2 plugin binary
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
-o /usr/local/lib/docker/cli-plugins/docker-compose

# Make it executable
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# Test it
docker compose version
```
For ARM (Graviton) instances, replace ```x86_64``` with ```aarch64```.

After that, create the following directory and navigate into it:

```bash
mkdir restaurant-app
cd restaurant-app
```

## Step 2: Set Up Your Project Structure

Inside ```restaurant-app```, create folders and files:

```bash
mkdir web
mkdir db
```

Create the following files in the appropriate directories as per the Project structure:

- ```docker-compose.yml```
- ```web/Dockerfile```
- ```web/app.py```
- ```web/requirements.txt```
- ```web/index.html```
- ```db/init.sql```

## Step 3: Write docker-compose.yml

This will define and connect your web app and database.

```yaml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "5000:5000"
    depends_on:
      - db
    environment:
      - DB_HOST=db
      - DB_USER=root
      - DB_PASSWORD=restaurant
      - DB_NAME=orders

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: restaurant
      MYSQL_DATABASE: orders
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  db_data:
```

## Step 4: Write ```web/Dockerfile```

Build your Flask app image:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]
```

## Step 5: Write ```web/requirements.txt```

Specify the Python packages:

```nginx
flask
mysql-connector-python
```

## Step 6: Write web/app.py

Your Flask backend serving the page and handling orders.

```python
from flask import Flask, request, redirect, url_for
import mysql.connector
import os

app = Flask(__name__, static_url_path='/static', static_folder='static')

@app.route('/')
def home():
    return open("index.html").read()

@app.route('/order', methods=['POST'])
def order():
    food = request.form['food']

    # Connect to the database
    db = mysql.connector.connect(
        host=os.environ['DB_HOST'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME']
    )
    cursor = db.cursor()
    cursor.execute("INSERT INTO orders (food) VALUES (%s)", (food,))
    db.commit()
    cursor.close()
    db.close()

    # Redirect using PRG pattern
    return redirect(url_for('order_received', food=food))

@app.route('/order-received')
def order_received():
    food = request.args.get('food', 'your meal')
    return f"Order received for: {food}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Step 7: Write web/index.html
Interactive page with image preview.

```html
<!DOCTYPE html>
<html>
<head>
  <title>African Restaurant</title>
  <link rel="stylesheet" href="/static/styles.css">
</head>
<body>
  <h1>Welcome to African Restaurant</h1>
  
  <form action="/order" method="POST">
    <label for="food">Choose your food:</label><br>
    <select name="food" id="food" onchange="showImage()">
      <option value="">--Select--</option>
      <option value="Eru">Eru</option>
      <option value="Egusi Soup">Egusi Soup</option>
      <option value="Bunny Chow">Bunny Chow</option>
    </select>
    <br><br>
    <img id="foodImage" src="" alt="Food Preview" style="display:none; width:300px; height:auto; border-radius:10px;">
    <br><br>
    <button type="submit">Order</button>
  </form>

  <script>
    function showImage() {
      const food = document.getElementById('food').value;
      const image = document.getElementById('foodImage');
      let imagePath = '';

      if (food === 'Eru') {
        imagePath = '/static/images/eru.jpg';
      } else if (food === 'Egusi Soup') {
        imagePath = '/static/images/egusi.jpg';
      } else if (food === 'Bunny Chow') {
        imagePath = '/static/images/bunny.jpg';
      }

      if (imagePath) {
        image.src = imagePath;
        image.style.display = 'block';
      } else {
        image.style.display = 'none';
      }
    }
  </script>
</body>
</html>
```

## Step 8: Initialize the Database - db/init.sql
```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    food VARCHAR(255)
);
```


After creating all the folders and files in the Ec2 instnance, do the following

## Step 9: Add Your Images in S3

### Create an S3 Bucket

- Go to AWS S3 Console

- Give it a name like ```african-restaurant-assets```

- Enable public access

- Create the bucket 

### Upload Your CSS file and Images

Upload your ```css``` file:

```css
body {
    font-family: Arial, sans-serif;
    background-color: #fff3e6;
    text-align: center;
    padding: 50px;
}

h1 {
    color: #cc5500;
}

form {
    margin-top: 30px;
}

#foodImage {
    margin-top: 20px;
    box-shadow: 0px 0px 10px rgba(0,0,0,0.3);
    border-radius: 10px;
}
```

Upload your Image files in a created folder called ```/images```:

- ```eru.jpg```

- ```egusi.jpg```

- ```bunny.jpg```

### Get the Public URLs

Once uploaded:

- Click on each image

- Copy the Object URL (e.g., ```https://african-restaurant-assets.s3.amazonaws.com/images/eru.jpg```)

### Update Your index.html JavaScript to Use S3 Links

Instead of:

```js
imagePath = '/static/images/eru.jpg';
```

Use:

```js
imagePath = 'https://african-restaurant-assets.s3.amazonaws.com/images/eru.jpg';
```

### Put a Bucket Policy

- Go to the bucket

- Go to the ```Permissions``` tab

- Under ```Block Public Access```, click Edit

- Uncheck all the blocking options

- Save changes

Then, add this bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```
Replace ```your-bucket-name``` with the actual name.


## Step 10: Build and Run the Project

Log back into the Instance and run the application.

From your restaurant-app folder:

```bash
sudo docker compose up --build
```

## Step 11: Access the App

- Open browser → ```http://localhost:5000```

- Pick a food — the image shows instantly.

- Click **Order** — your choice is saved in MySQL.

- See confirmation message.

## Step 12: Check the Orders in MySQL

Get into the MySQL container terminal:

```bash
docker exec -it restaurant-app-db-1 mysql -uroot -p 
```

Password: ```restaurant```

Run:

```sql
USE orders;
SELECT * FROM orders;
```

**NB**
You must attach an IAM role to your EC2 instance if your Flask app or any other component in your container is ever going to pull images from private S3 buckets programmatically.
But since you're using public S3 URLs in your frontend, technically, you don’t need a role if:
Your bucket is public. You're not using the AWS SDK or CLI inside your app.

However, if you ever plan to secure that bucket or use private links (signed URLs), then:

✅ Create an IAM role with AmazonS3ReadOnlyAccess.

✅ Attach it to your EC2 instance.
