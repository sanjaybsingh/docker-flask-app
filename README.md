# Docker Flask Application

This is a simple Flask application containerized using Docker. The application serves a "Hello, Docker!" message on port 5000.

## Project Structure

```
Docker-K8-practice/
│
├── app.py              # Flask application
├── requirements.txt    # Python dependencies
├── Dockerfile         # Docker configuration
└── README.md         # This file
```

## Application Details

### app.py
A simple Flask application that returns "Hello, Docker!" when accessed.

### requirements.txt
Contains the Python dependencies:
- Flask==3.0.0

### Dockerfile
```dockerfile
# Use an official Python runtime as the base image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container
COPY requirements.txt .

# Install the application dependencies
RUN pip install -r requirements.txt

# Copy the application code into the container
COPY app.py .

# Expose port 5000 for the Flask application
EXPOSE 5000

# Define the command to run the application
CMD ["python", "app.py"]
```

## How It Works

1. **Base Image**: Uses Python 3.9 slim image as the foundation
2. **Working Directory**: Sets `/app` as the working directory
3. **Dependencies**: 
   - Copies requirements.txt and installs dependencies
   - Uses Docker layer caching for efficient builds
4. **Application Code**: Copies app.py into the container
5. **Port**: Exposes port 5000
6. **Startup Command**: Runs the Flask application

## Building and Running

### Build the Docker Image
```bash
docker build -t flask-app .
```

### Run the Container
```bash
docker run -d -p 5000:5000 --name flask-test flask-app
```

## Testing the Application

There are several ways to test the application:

1. **Using PowerShell**:
```powershell
Invoke-WebRequest -Uri http://localhost:5000 -Method GET
```

2. **Web Browser**:
   - Open http://localhost:5000 in your browser

3. **View Container Logs**:
```bash
docker logs flask-test
```

4. **Check Container Status**:
```bash
docker ps
```

## Cleanup

To stop and remove the container:
```bash
docker stop flask-test
docker rm flask-test
```

## Best Practices Used

1. Using a specific Python version instead of `latest`
2. Copying requirements separately to leverage Docker cache
3. Using a slim base image to reduce size
4. Setting a working directory for better organization
5. Using `CMD` in exec form `["command", "arg1"]`

## Development Notes

- The application runs in development mode and should not be used in production without proper WSGI server configuration
- The container exposes port 5000 and maps it to port 5000 on the host machine
- The application is configured to listen on all interfaces (`0.0.0.0`) for container networking