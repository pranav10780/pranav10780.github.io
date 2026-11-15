---
title: Nasa space app hackathon
description: "My submission for nasa space apps hackathon"
date: 2026-11-15
tags: ["hackathon", "web", "python"]
---

## Introduction

Andromia health co-pilot is a small project i made for 2026 NASA Space Apps Challenge.

It was a little effort project which i decided to build, i wanted to join a team but unfortunately i couldn't so my front end engineer ended up being gemini (argon) model, which i have to say i am very impressed by since it did the entire heavy lifting on the fornt end side while i did the back.

For the backend i decided to use python with the flask framework for the api along with sqlite3 as the data base
The front end is html,css,js made by gemini i am not going over the front end as i don't really know how it works.

## The code

```python
def get_db_connection():
    conn = sqlite3.connect('local_health.db')
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db_connection()
    conn.execute('''
        CREATE TABLE IF NOT EXISTS health_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
            systolic_bp INTEGER,
            diastolic_bp INTEGER,
            effective_dose REAL,
            ucla_1 INTEGER,
            ucla_2 INTEGER,
            ucla_3 INTEGER,
            t_score REAL,
            sleep_hours REAL
        );
    ''')
    conn.commit()
    conn.close()
```

The `get_db_connection()` is an helper function which connects to the `local_health.db` file.

We use `sqlite3.Row` here because then we can access the elements like: 
`row['username']` instead of `row[1]` which helps escape headaches.

The `init_db()` function is used to setup/create the database table, the different values are:
- `id` is automatically assigned to help select specific records
- `timestamp` automatically generates a current UTC date and time
- `systolic_bp` and `diastolic_bp` are blodd pressure readings
- `effective_dose` is the amount of radiation exposed in mSv
- `ucla_1` `ucla_2` `ucla_3` are used to access the UCLA Lonliness Scale
- `t_score` is used to measure the bone density
- `sleep_hours` is self explanatory

```python
@app.route('/api/log', methods=['POST'])
def log_health_data():
    data = request.get_json()
    
    conn = get_db_connection()
    conn.execute('''
        INSERT INTO health_logs (
            systolic_bp, diastolic_bp, effective_dose, 
            --snip--
    ))
    conn.commit()
    conn.close()

    return jsonify({"status": "success", "message": "Health data logged successfully"}), 201
```

The flask api looks for requests to `/api/log` using the http `POST` method.

```js
            const payload = {
                systolic_bp: parseInt(document.getElementById('sys').value),
                --snip--
            };
            await fetch('/api/log', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });
```

This is the front end code that sends the `POST` request to our api

As you can see we first take all the values submitted by the user in the form and combine it into a `payload` and send it.

>NOTE: The header `Content-Type` is required for sending json data.

Then our `log_health_data()` function inserts it into the database and send a 201 success message to the front end.

```python
@app.route('/api/health/latest', methods=['GET'])
def get_latest_health():
    conn = get_db_connection()
    result = conn.execute(
        'SELECT * FROM health_logs ORDER BY id DESC LIMIT 1'
    )
    row = result.fetchone()
    conn.close()

    if row is None:
        return jsonify({"error": "No data found"}), 404
        
    metrics = {
        "systolic_bp": row['systolic_bp'],
        --snip--
    }
    ucla_total = metrics['ucla_1'] + metrics['ucla_2'] + metrics['ucla_3']

    alerts=[]
    #diagnostics
    if metrics["systolic_bp"] > 130 or metrics["diastolic_bp"] > 85:
        alerts.append("Elevated blood pressure. Hydrate and run cabin environmental checks.")
    --snip--
    return jsonify({
        "metrics": metrics,
        "ucla_total": ucla_total,
        "alerts": alerts
    })
```

The flask api looks for requests to `/api/health/latest` using the http `GET` method.

Then we open an connection to the database and read ONLY one row. 

```sql
'SELECT * FROM health_logs ORDER BY id DESC LIMIT 1'
```

This is because this data should be the latest so we will take the last data entry

```python
if row is None:
    return jsonify({"error": "No data found"}), 404
```

This is a simple check to see if there is any data in the database to be served, else we send a 404 not found.

The `metrics` variable stores all the health information in json format

The `ucla_total` adds all the ucla results to assess lonlinesss

The `diagnostic` part is simply a if else treee to check if the vitals are in the normal range.

```python
return jsonify({
    "metrics": metrics,
    "ucla_total": ucla_total,
    "alerts": alerts
})
```

This simply returns all the data back as json.

```python
@app.route('/api/health/history', methods=['GET'])
def get_health_history():
    conn = get_db_connection()
    result = conn.execute('SELECT * FROM health_logs ORDER BY id ASC')
    rows = result.fetchall()
    conn.close()

    history = []
    for row in rows:
        history.append({
            "timestamp": row['timestamp'],
            --snip--
        })
    
    return jsonify(history)
```

The flask api looks for requests to `/api/health/history` using the http `GET` method.

We fetch all the data in the database in ascending order and add it to a list named `history` and return it.

```python
@app.route('/')
def serve_frontend():
    return app.send_static_file('index.html')

if __name__ == '__main__':
    app.run(port=8080,debug=True)
```

The `serve_frontedn()` send the `index.html` basically acting as out webserver

```python
if __name__ == '__main__':
    app.run(port=8080,debug=True)
```

This ensures that our program is actually begin run and not called as a dependecies and if it is called a seperate program, it opens a port on 8080 where it listen for traffic.

Thank you and have a nice day

## See also

- [Andromia Health Co-pilot](https://github.com/pranav10780/andromia) - this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
