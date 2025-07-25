# African-Restaurant-Web-App-Dockerized-with-Flask-MySQL
Containerised African restaurant web app built with Flask and MySQL. Users select dishes, see images instantly, and place orders stored persistently in MySQL. Docker Compose orchestrates the multi-container setup for easy deployment and data management.

## Project Structure

```csharp
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
    └── static/
        ├── styles.css
        └── images/
            ├── jollof.jpg
            ├── egusi.jpg
            └── bunny.jpg
```

## Step 1: Create Your Project Folder

In you AWS Account, Launch an Ec2 instance with instance type at least ```t3.medium``` with ports, ```8080```, ```80```.

SSH into the Instance:

```bash
mkdir restaurant-app
cd restaurant-app
```

## Step 2: Set Up Your Project Structure

Inside ```restaurant-app```, create folders and files:

```bash
mkdir web
mkdir web/static
mkdir web/static/images
mkdir db
```

Create the following files:

- ```docker-compose.yml```
- ```web/Dockerfile```
- ```web/app.py```
- ```web/requirements.txt```
- ```web/index.html```
- ```web/static/styles.css```
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
from flask import Flask, request
import mysql.connector
import os

app = Flask(__name__, static_url_path='/static', static_folder='static')

@app.route('/')
def home():
    return open("index.html").read()

@app.route('/order', methods=['POST'])
def order():
    food = request.form['food']
    db = mysql.connector.connect(
        host=os.environ['DB_HOST'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME']
    )
    cursor = db.cursor()
    cursor.execute("INSERT INTO orders (food) VALUES (%s)", (food,))
    db.commit()
    return "Order received for: " + food

if __name__ == '__main__':
    app.run(host='0.0.0.0')
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
      <option value="Jollof Rice">Jollof Rice</option>
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

      if (food === 'Jollof Rice') {
        imagePath = '/static/images/jollof.jpg';
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

## Step 8: Write Your CSS - web/static/styles.css
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

## Step 9: Initialize the Database - db/init.sql
```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    food VARCHAR(255)
);
```


After creating all the folders and files in the Ec2 instnance, do the following

## Step 10: Add Your Images in S3

### Create an S3 Bucket

- Go to AWS S3 Console

- Give it a name like ```african-restaurant-assets```

- Enable public access

- Create the bucket 

### Upload Your Images

Upload your files like:

- ```jollof.jpg```

- ```egusi.jpg```

- ```bunny.jpg```

Inside a folder like ```/images``` if you want.

### Get the Public URLs

Once uploaded:

- Click on each image

- Copy the Object URL (e.g., ```https://african-restaurant-assets.s3.amazonaws.com/images/jollof.jpg```)

### Update Your index.html JavaScript to Use S3 Links

Instead of:

```js
imagePath = '/static/images/jollof.jpg';
```

Use:

```js
imagePath = 'https://african-restaurant-assets.s3.amazonaws.com/images/jollof.jpg';
```

Place your food images inside ```web/static/images/``` folder:

```jollof.jpg```

```egusi.jpg```

```bunny.jpg```


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


## Step 11: Build and Run the Project

Log back into the Instance and run the application.

From your restaurant-app folder:

```bash
docker-compose up --build
```

## Step 12: Access the App

- Open browser → ```http://localhost:5000```

- Pick a food — the image shows instantly.

- Click **Order** — your choice is saved in MySQL.

- See confirmation message.

## Step 13: Check the Orders in MySQL

Get into the MySQL container terminal:

```bash
docker exec -it restaurant-app_db_1 mysql -uroot -p
```

Password: ```restaurant```

Run:

```sql
USE orders;
SELECT * FROM orders;
```