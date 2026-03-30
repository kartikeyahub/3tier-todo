Below are the **complete corrected script files** for the 3‑tier architecture. They include:

- **VM3 (Database)**: Install PostgreSQL 15 from the official repository.
- **VM2 (Backend)**: Node.js backend with `pg` driver and systemd service.
- **VM1 (Frontend)**: React app created in home directory, then moved to `/var/www`, with Nginx reverse proxy.

Replace the placeholder IP addresses and passwords before running.

---

## Script 1: VM3 – PostgreSQL Database

**File:** `setup-database-3tier.sh`

```bash
#!/bin/bash

# 3-Tier Database VM (VM3) Setup Script
set -e

# === CONFIGURATION (EDIT THESE) ===
DB_PASSWORD="StrongPassword123"   # Choose a strong password
# =================================

echo "Updating system and installing prerequisites..."
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates gnupg lsb-release ufw

echo "Adding official PostgreSQL 15 repository..."
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/pgdg.gpg > /dev/null
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

echo "Installing PostgreSQL 15..."
sudo apt update
sudo apt install -y postgresql-15 postgresql-contrib-15

echo "Configuring PostgreSQL..."
sudo -u postgres psql << EOF
CREATE DATABASE todos_db;
ALTER USER postgres WITH PASSWORD '$DB_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE todos_db TO postgres;
\c todos_db
CREATE TABLE IF NOT EXISTS todos (
    id SERIAL PRIMARY KEY,
    text VARCHAR(255) NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO todos (text, completed) VALUES
('Learn Docker with 3-tier architecture', false),
('Master Docker Compose', false),
('Build production-ready applications', false)
ON CONFLICT DO NOTHING;
EOF

echo "Allowing remote connections..."
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/15/main/postgresql.conf
echo "host    all             all             0.0.0.0/0               scram-sha-256" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf

sudo systemctl restart postgresql

echo "Configuring firewall..."
sudo ufw allow 5432/tcp
echo "y" | sudo ufw enable

echo "Database VM setup complete!"
echo "PostgreSQL is ready. Connect from backend VM using:"
echo "psql -h $(hostname -I | awk '{print $1}') -U postgres -d todos_db"
echo ""
echo "⚠️  IMPORTANT: Reboot this VM now to load the latest kernel if needed, then proceed to the backend VM."
```

---

## Script 2: VM2 – Backend (Node.js + PostgreSQL)

**File:** `setup-backend-3tier.sh`

```bash
#!/bin/bash

# 3-Tier Backend VM (VM2) Setup Script
set -e

# === CONFIGURATION (EDIT THESE) ===
DB_VM_IP="<VM3_PRIVATE_IP>"        # IP of the PostgreSQL VM
DB_PASSWORD="StrongPassword123"    # Must match the password set on DB VM
# =================================

# Get the actual non-root username
CURRENT_USER=$(who am i | awk '{print $1}')
if [ -z "$CURRENT_USER" ]; then
    CURRENT_USER=$USER
fi

echo "Updating system and installing Node.js 18.x..."
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs ufw

echo "Creating backend directory..."
sudo mkdir -p /var/www/todo-backend
sudo chown -R $CURRENT_USER:$CURRENT_USER /var/www/todo-backend
cd /var/www/todo-backend

echo "Creating package.json with PostgreSQL driver..."
cat > package.json << 'EOF'
{
  "name": "todo-backend-3tier",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^4.18.2", "cors": "^2.8.5", "pg": "^8.11.0" }
}
EOF

echo "Creating db.js..."
cat > db.js << EOF
const { Pool } = require('pg');
const pool = new Pool({
  host: '$DB_VM_IP',
  port: 5432,
  user: 'postgres',
  password: '$DB_PASSWORD',
  database: 'todos_db',
});
module.exports = { query: (text, params) => pool.query(text, params) };
EOF

echo "Creating server.js (PostgreSQL version)..."
cat > server.js << 'EOF'
const express = require('express');
const cors = require('cors');
const db = require('./db');

const app = express();
const PORT = process.env.PORT || 5000;

app.use(cors());
app.use(express.json());

app.get('/api/todos', async (req, res) => {
  try {
    const result = await db.query('SELECT * FROM todos ORDER BY id');
    res.json(result.rows);
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Database error' });
  }
});

app.post('/api/todos', async (req, res) => {
  const { text } = req.body;
  if (!text || !text.trim()) return res.status(400).json({ error: 'Text required' });
  try {
    const result = await db.query('INSERT INTO todos (text) VALUES ($1) RETURNING *', [text.trim()]);
    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Database error' });
  }
});

app.put('/api/todos/:id', async (req, res) => {
  const id = parseInt(req.params.id);
  try {
    const result = await db.query('UPDATE todos SET completed = NOT completed WHERE id = $1 RETURNING *', [id]);
    if (result.rows.length === 0) return res.status(404).json({ error: 'Not found' });
    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Database error' });
  }
});

app.delete('/api/todos/:id', async (req, res) => {
  const id = parseInt(req.params.id);
  try {
    const result = await db.query('DELETE FROM todos WHERE id = $1 RETURNING *', [id]);
    if (result.rows.length === 0) return res.status(404).json({ error: 'Not found' });
    res.json({ message: 'Deleted', todo: result.rows[0] });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Database error' });
  }
});

app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
EOF

echo "Installing dependencies..."
npm install

echo "Creating systemd service..."
sudo tee /etc/systemd/system/todo-backend.service > /dev/null << EOF
[Unit]
Description=Todo Backend 3-Tier
After=network.target

[Service]
Type=simple
User=$CURRENT_USER
WorkingDirectory=/var/www/todo-backend
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start todo-backend
sudo systemctl enable todo-backend

echo "Configuring firewall..."
sudo ufw allow 5000/tcp
echo "y" | sudo ufw enable

echo "Backend VM setup complete!"
echo "Backend API running on port 5000"
echo "Test locally: curl http://localhost:5000/api/todos"
echo "Test from frontend VM: curl http://$(hostname -I | awk '{print $1}'):5000/api/todos"
```

---

## Script 3: VM1 – Frontend (React + Nginx)

**File:** `setup-frontend-3tier.sh`

```bash
#!/bin/bash

# 3-Tier Frontend VM (VM1) Setup Script
set -e

# === CONFIGURATION (EDIT THESE) ===
BACKEND_VM_IP="<VM2_PRIVATE_IP>"   # IP of the backend VM (Node.js)
# =================================

# Get the actual non-root username
CURRENT_USER=$(who am i | awk '{print $1}')
if [ -z "$CURRENT_USER" ]; then
    CURRENT_USER=$USER
fi

echo "Updating system and installing prerequisites..."
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nginx ufw

echo "Installing Node.js 18.x..."
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

echo "Creating React app in user's home directory..."
cd /home/$CURRENT_USER
sudo rm -rf todo-frontend
sudo -u $CURRENT_USER npx create-react-app todo-frontend

echo "Moving React app to /var/www..."
sudo rm -rf /var/www/todo-frontend
sudo mv todo-frontend /var/www/
sudo chown -R $CURRENT_USER:$CURRENT_USER /var/www/todo-frontend

cd /var/www/todo-frontend

echo "Installing axios..."
npm install axios

echo "Configuring React app to use backend at $BACKEND_VM_IP..."
cat > src/App.js << 'EOF'
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = `http://BACKEND_IP_PLACEHOLDER:5000`;

function App() {
  const [todos, setTodos] = useState([]);
  const [newTodo, setNewTodo] = useState('');
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  useEffect(() => { fetchTodos(); }, []);

  const fetchTodos = async () => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_URL}/api/todos`);
      setTodos(response.data);
      setError('');
    } catch (err) {
      setError('Failed to fetch todos. Backend may be down.');
    } finally {
      setLoading(false);
    }
  };

  const addTodo = async () => {
    if (newTodo.trim()) {
      try {
        const response = await axios.post(`${API_URL}/api/todos`, { text: newTodo });
        setTodos([...todos, response.data]);
        setNewTodo('');
      } catch (err) {
        setError('Failed to add todo.');
      }
    }
  };

  const toggleTodo = async (id) => {
    try {
      const response = await axios.put(`${API_URL}/api/todos/${id}`);
      setTodos(todos.map(todo => todo.id === id ? response.data : todo));
    } catch (err) {
      setError('Failed to update todo.');
    }
  };

  const deleteTodo = async (id) => {
    try {
      await axios.delete(`${API_URL}/api/todos/${id}`);
      setTodos(todos.filter(todo => todo.id !== id));
    } catch (err) {
      setError('Failed to delete todo.');
    }
  };

  if (loading) return <div style={{ padding: '20px', textAlign: 'center' }}><h2>Loading todos...</h2></div>;

  return (
    <div style={{ padding: '20px', fontFamily: 'Arial', maxWidth: '600px', margin: '0 auto' }}>
      <h1>🐳 Docker Todo App - 3 Tier (Manual Setup)</h1>
      <p style={{ color: '#666' }}>Frontend + Backend + PostgreSQL on separate VMs</p>
      {error && <div style={{ padding: '10px', backgroundColor: '#f8d7da', color: '#721c24', borderRadius: '4px', marginBottom: '20px' }}>{error}</div>}
      <div style={{ margin: '20px 0' }}>
        <input
          type="text"
          value={newTodo}
          onChange={(e) => setNewTodo(e.target.value)}
          placeholder="Add a new todo..."
          style={{ padding: '10px', marginRight: '10px', width: '70%', border: '1px solid #ddd', borderRadius: '4px' }}
          onKeyPress={(e) => e.key === 'Enter' && addTodo()}
        />
        <button onClick={addTodo} style={{ padding: '10px 20px', backgroundColor: '#28a745', color: 'white', border: 'none', borderRadius: '4px', cursor: 'pointer' }}>Add Todo</button>
      </div>
      <div>
        {todos.map(todo => (
          <div key={todo.id} style={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between', padding: '10px', margin: '10px 0', backgroundColor: '#f8f9fa', borderRadius: '4px', border: '1px solid #e9ecef' }}>
            <div onClick={() => toggleTodo(todo.id)} style={{ cursor: 'pointer', flex: 1, textDecoration: todo.completed ? 'line-through' : 'none', color: todo.completed ? '#6c757d' : '#212529' }}>
              {todo.text}
            </div>
            <button onClick={() => deleteTodo(todo.id)} style={{ padding: '5px 10px', backgroundColor: '#dc3545', color: 'white', border: 'none', borderRadius: '4px', cursor: 'pointer', marginLeft: '10px' }}>Delete</button>
          </div>
        ))}
      </div>
    </div>
  );
}

export default App;
EOF

sed -i "s|BACKEND_IP_PLACEHOLDER|$BACKEND_VM_IP|g" src/App.js

echo "Building React app..."
npm run build

echo "Deploying to Nginx..."
sudo rm -rf /var/www/html/*
sudo cp -r build/* /var/www/html/

sudo tee /etc/nginx/sites-available/todo-app > /dev/null << EOF
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.html;
    location / {
        try_files \$uri \$uri/ /index.html;
    }
    location /api {
        proxy_pass http://$BACKEND_VM_IP:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/todo-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl restart nginx

sudo ufw allow 80/tcp
echo "y" | sudo ufw enable

echo "Frontend VM setup complete!"
echo "Access the app at http://$(hostname -I | awk '{print $1}')"
```

---

## Instructions for Use

1. **Copy each script** to the corresponding VM.
2. **Edit the configuration variables** at the top of each script (IP addresses and password).
3. **Make each script executable**:
   ```bash
   chmod +x setup-*.sh
   ```
4. **Run the scripts in order** (as `sudo`):
   - First: `sudo ./setup-database-3tier.sh` on VM3
   - Second: `sudo ./setup-backend-3tier.sh` on VM2
   - Third: `sudo ./setup-frontend-3tier.sh` on VM1

5. **After the database script finishes**, reboot VM3 if the kernel update message appears:
   ```bash
   sudo reboot
   ```

6. **Test connectivity**:
   - From VM2: `curl http://localhost:5000/api/todos`
   - From VM1: `curl http://<VM2_IP>:5000/api/todos`
   - Open browser: `http://<VM1_IP>`

All scripts are now ready to use.


Great! Now that your 3‑tier application is running, here are several ways to **check database updates** (new todos, changes, deletions) in real time or on demand.

---

## 1. Direct SQL Query (most reliable)

SSH into your **database VM (VM3)** and run:

```bash
sudo -u postgres psql -d todos_db -c "SELECT * FROM todos ORDER BY id DESC;"
```

**To watch for changes every 2 seconds:**  
```bash
watch -n 2 'sudo -u postgres psql -d todos_db -c "SELECT id, text, completed, created_at FROM todos ORDER BY id DESC LIMIT 10;"'
```

**Interactive PostgreSQL shell (live queries):**  
```bash
sudo -u postgres psql -d todos_db
```
Then inside the shell:
```sql
SELECT * FROM todos ORDER BY id DESC;
-- Repeat with \watch 2
SELECT id, text, completed, created_at FROM todos ORDER BY id DESC LIMIT 5;
\watch 2
```

---

## 2. Query from Backend VM (VM2)

From the backend VM, you can also query the database using `psql` if you install the client:

```bash
# Install PostgreSQL client on backend VM (optional)
sudo apt install -y postgresql-client

# Then connect to the database VM
psql -h <VM3_IP> -U postgres -d todos_db -W
# Enter the password you set (e.g., StrongPassword123)
```

Then run the same SQL queries.

---

## 3. Use the Backend API (from any VM)

### Get all todos:
```bash
curl -s http://<VM2_IP>:5000/api/todos | jq .
```
If `jq` is not installed: `sudo apt install jq -y`

### Watch API changes in real time:
```bash
watch -n 2 'curl -s http://<VM2_IP>:5000/api/todos | jq ".[] | {id, text, completed}"'
```

### Get only the latest todo:
```bash
curl -s http://<VM2_IP>:5000/api/todos | jq '.[-1]'
```

---

## 4. Monitor PostgreSQL Logs for INSERT/UPDATE/DELETE

On the database VM (VM3), enable query logging (temporary for debugging):

```bash
sudo sed -i "s/log_statement = 'none'/log_statement = 'all'/" /etc/postgresql/15/main/postgresql.conf
sudo systemctl restart postgresql
```

Then watch the logs:
```bash
sudo tail -f /var/log/postgresql/postgresql-15-main.log | grep -E "INSERT|UPDATE|DELETE|SELECT"
```

To revert after testing:
```bash
sudo sed -i "s/log_statement = 'all'/log_statement = 'none'/" /etc/postgresql/15/main/postgresql.conf
sudo systemctl restart postgresql
```

---

## 5. Real‑time Monitoring Script (on any VM)

Create a simple bash script that polls the API and shows new entries:

```bash
cat > ~/monitor-todos.sh << 'EOF'
#!/bin/bash

API_URL="http://<VM2_IP>:5000/api/todos"
LAST_COUNT=0

echo "Monitoring todos at $API_URL (press Ctrl+C to stop)"
while true; do
    RESPONSE=$(curl -s "$API_URL")
    CURRENT_COUNT=$(echo "$RESPONSE" | jq 'length')
    if [ "$CURRENT_COUNT" -gt "$LAST_COUNT" ] && [ "$LAST_COUNT" -ne 0 ]; then
        NEW=$((CURRENT_COUNT - LAST_COUNT))
        echo "[$(date '+%H:%M:%S')] ➕ $NEW new todo(s) added!"
        echo "$RESPONSE" | jq ".[-${NEW}] | {id, text, created_at}"
        echo "---"
    elif [ "$CURRENT_COUNT" -lt "$LAST_COUNT" ]; then
        echo "[$(date '+%H:%M:%S')] ❌ Todo(s) deleted (was $LAST_COUNT, now $CURRENT_COUNT)"
    fi
    LAST_COUNT=$CURRENT_COUNT
    sleep 2
done
EOF

chmod +x ~/monitor-todos.sh
# Edit the API_URL inside the script (replace <VM2_IP>)
nano ~/monitor-todos.sh
./monitor-todos.sh
```

---

## 6. Simple Web UI Check

Just open your frontend URL (`http://<VM1_IP>`) and add/delete todos. Every change is immediately stored in PostgreSQL. You can verify by refreshing the SQL query in another terminal.

---

## Quick Summary Table

| Method | Command (run on appropriate VM) |
|--------|--------------------------------|
| **SQL – one‑time** | `sudo -u postgres psql -d todos_db -c "SELECT * FROM todos;"` (on VM3) |
| **SQL – watch** | `watch -n 2 'sudo -u postgres psql -d todos_db -c "SELECT id, text, completed FROM todos ORDER BY id DESC LIMIT 5;"'` |
| **API – one‑time** | `curl -s http://<VM2_IP>:5000/api/todos \| jq .` |
| **API – watch** | `watch -n 2 'curl -s http://<VM2_IP>:5000/api/todos \| jq ".[] \| {id, text}"'` |
| **PostgreSQL log tail** | `sudo tail -f /var/log/postgresql/postgresql-15-main.log \| grep -E "INSERT|UPDATE|DELETE"` (on VM3) |

Choose the method that fits your workflow. For quick debugging, the SQL watch is most direct; for integration testing, the API watch is better.
