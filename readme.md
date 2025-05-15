# **Docker-Based JavaScript Application with MongoDB Backend**

## **Objective**

Create a JavaScript-based web application running in Docker containers that allows users to enter personal data (First Name, Last Name, Age, City) and save it in a MongoDB NoSQL database. After submitting data, a file should be generated displaying all stored entries.

## **Project Structure**

- **Frontend (Node.js & Express + Basic HTML Form)** - Allows users to input and check stored data.
- **Backend (MongoDB)** - Stores the submitted data.
- **Docker Compose** - Manages both containers.

## **Installation & Deployment**

### **1. Install Docker & Docker Compose**

Ensure Docker and Docker Compose are installed on your system.

```sh
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

### **2. Create the Project Directory**

```sh
mkdir docker-js-mongo-app && cd docker-js-mongo-app
```

### **3. Create ****`docker-compose.yml`**

In the project root, create the `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  mongo:
    image: mongo:latest
    container_name: mongo_db
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  frontend:
    build: ./frontend
    container_name: js_app
    restart: always
    ports:
      - "3000:3000"
    environment:
      - SERVER_HOST=0.0.0.0
    depends_on:
      - mongo

volumes:
  mongo_data:
```

### **4. Create Frontend Application**

#### **4.1 Create ****`frontend/Dockerfile`**
```bash
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
EXPOSE 3000
```

#### **4.2 Create ****`frontend/package.json `**

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.2.1",
    "cors": "^2.8.5"
  }
}
```

#### **4.3 Create ****`frontend/server.js`**

```js
const express = require('express');
const mongoose = require('mongoose');
const fs = require('fs');
const cors = require('cors');
const path = require('path');

const app = express();
app.use(express.json());
app.use(cors());
app.use(express.static(path.join(__dirname, 'public')));

mongoose.connect('mongodb://mongo:27017/mydatabase', {
    useNewUrlParser: true,
    useUnifiedTopology: true
});

const UserSchema = new mongoose.Schema({
    firstName: String,
    lastName: String,
    age: Number,
    city: String
});
const User = mongoose.model('User', UserSchema);

app.post('/submit', async (req, res) => {
    try {
        console.log("Received data:", req.body);
        const newUser = new User(req.body);
        await newUser.save();
        res.json({ message: 'User data saved successfully!' });
    } catch (error) {
        console.error("Error saving data:", error);
        res.status(500).json({ message: 'Error saving data', error });
    }
});

app.get('/data', async (req, res) => {
    try {
        const users = await User.find();
        res.json(users);
    } catch (error) {
        res.status(500).json({ message: 'Error retrieving data', error });
    }
});

app.get('/generate-file', async (req, res) => {
    try {
        console.log("Generating user file...");
        const users = await User.find();
        const filePath = path.join(__dirname, 'public', 'users.json');
        fs.writeFileSync(filePath, JSON.stringify(users, null, 2));
        console.log("File saved at:", filePath);
        res.download(filePath);
    } catch (error) {
        console.error("Error generating file:", error);
        res.status(500).json({ message: 'Error generating file', error });
    }
});

app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

const PORT = 3000;
const HOST = '0.0.0.0';
app.listen(PORT, HOST, () => console.log(`Frontend running on http://${HOST}:${PORT}`));
```

#### **4.4 Create ****`frontend/public/index.html`**

```html
<!DOCTYPE html>
<html>
<head>
    <title>User Form</title>
    <script>
        const apiUrl = window.location.origin;

        async function submitData() {
            const data = {
                firstName: document.getElementById('firstName').value,
                lastName: document.getElementById('lastName').value,
                age: parseInt(document.getElementById('age').value),
                city: document.getElementById('city').value
            };
            await fetch(`${apiUrl}/submit`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(data)
            });
            loadData();
        }

        async function loadData() {
            const response = await fetch(`${apiUrl}/data`);
            const data = await response.json();
            const tableBody = document.getElementById('dataTableBody');
            tableBody.innerHTML = '';
            data.forEach(user => {
                const row = `<tr>
                    <td>${user.firstName}</td>
                    <td>${user.lastName}</td>
                    <td>${user.age}</td>
                    <td>${user.city}</td>
                </tr>`;
                tableBody.innerHTML += row;
            });
        }

        function checkData() {
            window.open(`${apiUrl}/generate-file`, '_blank');
        }
    </script>
</head>
<body>
    <h2>Enter User Information</h2>
    <input type="text" id="firstName" placeholder="First Name">
    <input type="text" id="lastName" placeholder="Last Name">
    <input type="number" id="age" placeholder="Age">
    <input type="text" id="city" placeholder="City">
    <button onclick="submitData()">Submit</button>
    
    <h3>Stored Data</h3>
    <table border="1">
        <thead>
            <tr>
                <th>First Name</th>
                <th>Last Name</th>
                <th>Age</th>
                <th>City</th>
            </tr>
        </thead>
        <tbody id="dataTableBody"></tbody>
    </table>
    
    <h3>Download Data</h3>
    <button onclick="checkData()">Download JSON File</button>
    
    <script>
        loadData();
    </script>
</body>
</html>
```

### **5. Restart the Application**

```sh
docker-compose down && docker-compose up --build -d
```

### **6. Expected Outcome**

- The front-end form collects user data and stores it in MongoDB.
- Users can view stored data in a table format below the form.
- Users can download a JSON file of the stored data.
- The app runs in Docker containers.
- API requests now dynamically use the correct server IP, preventing `localhost` issues.



# Task 2

# **Docker-Based JavaScript Application with MySql Backend**

- Modify your Docker Compose file, server.js, and index.html to replace MongoDB with MySQL and allow downloading the SQL output instead of a JSON file.

✅ Step 1: Update docker-compose.yml

-  Modify docker-compose.yml to use MySQL instead of MongoDB.

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:latest
    container_name: mysql_db
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mydatabase
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - mysql_data:/var/lib/mysql

  frontend:
    build: ./frontend
    container_name: js_app
    restart: always
    ports:
      - "<the_port_expose>:3000"
    environment:
      - DB_HOST=mysql
      - DB_USER=user
      - DB_PASSWORD=password
      - DB_NAME=mydatabase
    depends_on:
      - mysql

volumes:
  mysql_data:
```

- Replaces MongoDB with MySQL.
- Sets MySQL credentials (user:password).
- Exposes MySQL on port 3306.

✅ Step 2: Update server.js

- Modify server.js to work with MySQL instead of MongoDB.

example:

```js
const express = require('express');
const mysql = require('mysql2');
const fs = require('fs');
const cors = require('cors');
const path = require('path');

const app = express();
app.use(express.json());
app.use(cors());
app.use(express.static(path.join(__dirname, 'public')));

// Database connection
const db = mysql.createConnection({
    host: 'mysql',
    user: 'user',
    password: 'password',
    database: 'mydatabase'
});

db.connect(err => {
    if (err) {
        console.error('MySQL connection error:', err);
    } else {
        console.log('Connected to MySQL database');
        db.query(`CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            firstName VARCHAR(50),
            lastName VARCHAR(50),
            age INT,
            city VARCHAR(50)
        )`, (err) => {
            if (err) console.error("Table creation error:", err);
        });
    }
});

// Insert user data
app.post('/submit', (req, res) => {
    const { firstName, lastName, age, city } = req.body;
    const sql = "INSERT INTO users (firstName, lastName, age, city) VALUES (?, ?, ?, ?)";
    db.query(sql, [firstName, lastName, age, city], (err, result) => {
        if (err) {
            console.error("Insert error:", err);
            res.status(500).json({ message: 'Error saving data', error: err });
        } else {
            res.json({ message: 'User data saved successfully!' });
        }
    });
});

// Retrieve data
app.get('/data', (req, res) => {
    db.query("SELECT * FROM users", (err, results) => {
        if (err) {
            res.status(500).json({ message: 'Error retrieving data', error: err });
        } else {
            res.json(results);
        }
    });
});

// Generate SQL file and allow download
app.get('/generate-file', (req, res) => {
    db.query("SELECT * FROM users", (err, results) => {
        if (err) {
            res.status(500).json({ message: 'Error generating file', error: err });
        } else {
            let sqlDump = "INSERT INTO users (firstName, lastName, age, city) VALUES \n";
            sqlDump += results.map(row => `('${row.firstName}', '${row.lastName}', ${row.age}, '${row.city}')`).join(",\n") + ";";

            const filePath = path.join(__dirname, 'public', 'users.sql');
            fs.writeFileSync(filePath, sqlDump);
            res.download(filePath);
        }
    });
});

// Serve frontend
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

const PORT = 3000;
const HOST = '0.0.0.0';
app.listen(PORT, HOST, () => console.log(`Frontend running on http://${HOST}:${PORT}`));
```

✅ Step 3: Update index.html

- Modify the form and download button.

```html
<!DOCTYPE html>
<html>
<head>
    <title>User Form</title>
    <script>
        const apiUrl = window.location.origin;

        async function submitData() {
            const data = {
                firstName: document.getElementById('firstName').value,
                lastName: document.getElementById('lastName').value,
                age: parseInt(document.getElementById('age').value),
                city: document.getElementById('city').value
            };
            await fetch(`${apiUrl}/submit`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(data)
            });
            loadData();
        }

        async function loadData() {
            const response = await fetch(`${apiUrl}/data`);
            const data = await response.json();
            const tableBody = document.getElementById('dataTableBody');
            tableBody.innerHTML = '';
            data.forEach(user => {
                const row = `<tr>
                    <td>${user.firstName}</td>
                    <td>${user.lastName}</td>
                    <td>${user.age}</td>
                    <td>${user.city}</td>
                </tr>`;
                tableBody.innerHTML += row;
            });
        }

        function downloadSQL() {
            window.open(`${apiUrl}/generate-file`, '_blank');
        }
    </script>
</head>
<body>
    <h2>Enter User Information</h2>
    <input type="text" id="firstName" placeholder="First Name">
    <input type="text" id="lastName" placeholder="Last Name">
    <input type="number" id="age" placeholder="Age">
    <input type="text" id="city" placeholder="City">
    <button onclick="submitData()">Submit</button>
    
    <h3>Stored Data</h3>
    <table border="1">
        <thead>
            <tr>
                <th>First Name</th>
                <th>Last Name</th>
                <th>Age</th>
                <th>City</th>
            </tr>
        </thead>
        <tbody id="dataTableBody"></tbody>
    </table>
    
    <h3>Download SQL File</h3>
    <button onclick="downloadSQL()">Download SQL File</button>
    
    <script>
        loadData();
    </script>
</body>
</html>

```

- Replaces JSON download with SQL file download or you can modify to get output with sql output as tables in txt format.
- Calls /generate-file to generate a SQL INSERT statement.


✅ Step 4: Install Dependencies
Make sure you install the MySQL driver for Node.js:
```
npm install express mysql2 cors fs path
```
✅ Step 5: Configure your nginx to serve the javascript app

🎯 Summary

    **Step**	 **Action**
    - Step 1	Replaced MongoDB with MySQL in docker-compose.yml
    - Step 2	Updated server.js to use MySQL
    - Step 3	Modified index.html to download SQL instead of JSON
    - Step 4	Installed MySQL dependencies
    - Step 5	Ran docker-compose up -d
    - Step 6	Accessed http://localhost:3000
    - Step 7    Configure your nginx (Note not docker nginx)

